+++
title = "Transactional Outbox Pattern: Keeping the Database and Broker in Sync"
date = 2026-07-09

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ", "MassTransit", "outbox", "messaging", "EF Core", "distributed systems"]
+++

The transactional outbox pattern is easiest to understand from the failure it prevents.

An agent sends a reply in a chat system. The API stores the reply in SQL Server, returns `200 OK`, and the agent sees the message in the dashboard. But the customer never receives it because the process died before publishing the RabbitMQ event that would call WhatsApp.

The database says the reply exists. RabbitMQ never received the message. The consumer never ran. No error. No alert.

That is the dual-write problem: one request tries to update two independent systems, the database and the broker, but they do not share one transaction. At scale, "almost always works" (say 99.9%) is still a silent data-integrity bug — unacceptable for customer-visible or financial events.

---

## 1. The unsafe version

The first implementation people usually write is:

```csharp
session.AddOutboundMessage(text);
await dbContext.SaveChangesAsync();
await publishEndpoint.Publish(new MessageCreated(...));
```

This looks fine on the happy path, but it contains a dangerous gap.

```text
SaveChangesAsync() commits to SQL
    |
    |  process crash, broker outage, timeout, pod restart
    v
Publish(...) never reaches RabbitMQ
```

After that gap:

| System | State |
|--------|-------|
| SQL Server | The business row exists |
| RabbitMQ | The integration event is missing |
| Consumer | Never runs |

This is a **lost event**.

Reversing the order does not fix it:

```csharp
await publishEndpoint.Publish(new MessageCreated(...));
await dbContext.SaveChangesAsync();
```

Now a consumer can observe an event for a database change that later rolls back. That is a **phantom event**.

| Order | Failure window | Result |
|-------|----------------|--------|
| DB first, then publish | Commit succeeds, publish fails | Lost event |
| Publish first, then DB | Publish succeeds, commit fails | Phantom event |

The problem is not the order. The problem is trying to make two independent systems behave like one atomic write.

### Two hops, only one is fragile

It helps to split the path:

```text
[app commits to SQL] ── hop A ──▶ [message on RabbitMQ] ── hop B ──▶ [consumer runs]
```

- **Hop B (broker → consumer)** is already reliable enough in practice. MassTransit retries, redelivers, and dead-letters. You do not need an outbox for hop B.
- **Hop A (app → broker)** has no transactional guarantee. `Publish` is a network call to a separate system, not part of the DB transaction. That is where events vanish silently.

So when people say "the integration handler sometimes does not fire," the handler usually *would* run 100% of the time **if the message reached the broker**. The real bug is that the message sometimes never reaches the broker.

Typical hop-A failure modes (all leave the DB committed with nothing on the bus):

1. Broker down or unreachable — `Publish` throws.
2. Publish times out or the connection drops mid-call.
3. Process dies in the gap after commit (pod eviction, deploy, OOM) before publish.
4. Publish "succeeds" locally but the message never lands durably (client buffer lost, broker crash before flush).

---

## 2. The outbox idea

Instead of publishing to RabbitMQ inside the request, write the business change and a "publish this later" record in the **same database transaction**.

```sql
BEGIN TRANSACTION;

INSERT INTO SessionMessages (...);

INSERT INTO OutboxMessages
    (Id, MessageType, Payload, CorrelationId, OccurredAt, ProcessedAt)
VALUES
    (..., 'DispatchMessageCreated', '{...}', ..., SYSUTCDATETIME(), NULL);

COMMIT;
```

Now the commit has only two possible outcomes:

| Commit result | Business row | Outbox row |
|---------------|--------------|------------|
| Success | Present | Present |
| Failure / rollback | Absent | Absent |

There is no state where the message is saved but the intent to publish is missing.

### Why is the DB write "safe" if the DB is also external?

The outbox write is **not a second independent write**. It is part of the *same* transaction as the business change. That converts partial failure into all-or-nothing.

| Pattern | Writes | Middle state if one fails |
|---------|--------|---------------------------|
| Direct publish | SQL + RabbitMQ (two systems) | Yes — state saved, event lost (or phantom) |
| Outbox | SQL only (business row + outbox row) | No — both present or both rolled back |

If the commit fails, the business row did not save either. The command returns an error, the caller retries, and nothing half-happened. Even a crash mid-commit is safe under SQL Server WAL: the transaction is fully applied or fully discarded.

RabbitMQ is still not updated at commit time. That is intentional. After commit, a background **relay** reads the outbox table and publishes with retries. Failures on that path cause **delay**, not **loss**: publish fails → row stays unprocessed → retried; crash after publish before "mark processed" → republished (duplicate) → consumer dedupes.

The outbox row turns "I hope this process survives long enough to publish" into "the database durably records that this event must be published."

In a multi-context system (several `DbContext`s / bounded contexts), EF Core's built-in single-context outbox often does not fit. Each context typically owns its own `OutboxMessage` table and its own relay.

---

## 3. What an outbox row contains

An `OutboxMessage` row is a durable publish job.

| Column | Purpose |
|--------|---------|
| `Id` | Row identity |
| `MessageType` | CLR type name used by the relay to deserialize the payload |
| `Payload` | Serialized integration event |
| `CorrelationId` | Connects related work across hops and helps dedupe |
| `OccurredAt` | When the event was staged (used for claim order) |
| `ProcessedAt` | `NULL` until the relay successfully publishes |
| `Attempts` | Number of row-level failures |
| `FaultedAt` | Set when the row is parked as poison |
| `LockedBy`, `LockedAt`, `LockedUntil` | Lease fields so multiple relay instances can coordinate |

Before the relay publishes:

```text
Id | MessageType              | ProcessedAt | Attempts
1  | DispatchMessageCreated   | NULL        | 0
```

After a successful publish:

```text
Id | MessageType              | ProcessedAt                | Attempts
1  | DispatchMessageCreated   | 2026-07-09 10:00:07 +00:00 | 0
```

### Mark-then-sweep (not delete-on-publish)

On success the relay does **not** delete the row. It marks it:

```text
staged       ProcessedAt = NULL, FaultedAt = NULL   ← claimable
    │  relay publishes OK
    v
processed    ProcessedAt = <timestamp>              ← stays, no longer claimable
    │  ~7 days later
    v
deleted      retention pass removes it
```

| State | `ProcessedAt` | `FaultedAt` | In table? | Deleted? |
|-------|:-------------:|:-----------:|:---------:|----------|
| Pending | NULL | NULL | yes | — |
| Published OK | set | NULL | yes | after retention (e.g. 7 days) |
| Fault-parked | NULL | set | yes | **never** auto (operator only) |

Why mark-then-sweep instead of delete-on-publish:

- Publish path stays a cheap single-column update.
- Batched deletes avoid lock/index churn on the hot path.
- A short audit window answers "was this published, and when?"

Faulted rows are intentionally left for operators. Auto-deleting poison messages hides the problem. (They also keep payload data — something ops and privacy reviews must account for.)

---

## 4. Where the row is created

In the implementation I studied, command handlers do not manually insert outbox rows everywhere. The flow is built around domain events and EF Core `SaveChanges`.

The "before" shape was a **post-commit** domain event handler (or command code) that injected `IPublishEndpoint` and published directly — no durable record of intent. The "after" shape stages a row in the same transaction:

```csharp
// Before: direct publish after commit — lossy if process dies in the gap
await publishEndpoint.Publish(new DispatchInboundSessionMessageAdded(...));

// After: stage in the same SaveChanges transaction; relay publishes later
integrationEventOutbox.Add(new DispatchInboundSessionMessageAdded(...));
```

```text
Command handler
  |
  | aggregate mutation raises a domain event in memory
  v
SaveChangesAsync()
  |
  | PRE-COMMIT: DomainEventPublishInterceptor runs transactional handlers
  |   |
  |   v
  | ITransactionalDomainEventHandler<T>
  |   |
  |   v
  | integrationEventOutbox.Add(integrationEvent)
  |
  | EF inserts business rows + OutboxMessage rows
  v
COMMIT
  |
  v
POST-COMMIT handlers: SignalR, cache refresh, other non-transactional side effects
```

The important detail is that `integrationEventOutbox.Add(...)` does not publish anything. It is just an EF tracked insert:

```csharp
public void Add(IIntegrationEvent integrationEvent)
    => dbContext
        .Set<OutboxMessage>()
        .Add(OutboxMessage.Create(integrationEvent));
```

"Handling" the outbox message therefore splits into two moments, and neither is an "integration-event publish interceptor":

| Stage | When | By what |
|-------|------|---------|
| **Persisted** | Synchronously, at commit of the triggering `SaveChangesAsync` | Ordinary EF change tracking after `Add` |
| **Published** | Asynchronously, on the next relay tick (often 0–5s) | `OutboxRelayService` background poller |

There is no `IntegrationEventPublishInterceptor` by design. Domain events are dispatched *by* an interceptor (`DomainEventPublishInterceptor`); integration events are *not*. An interceptor that published on commit would reintroduce the crash-in-the-gap problem the outbox exists to remove.

Rules that keep staging safe:

- A transactional handler **never** calls `SaveChangesAsync` itself. It only `Add`s and rides the triggering save.
- If a transactional handler throws, the whole save fails — business state and publish intent succeed or fail together.
- Integration events that other contexts must receive → pre-commit transactional handlers.
- SignalR / cache / UI → post-commit handlers (must never roll back a successful commit).

| Work | Phase | Reason |
|------|-------|--------|
| Integration event that other contexts must receive | Pre-commit transactional handler | Must commit atomically with business state |
| SignalR push, cache refresh, UI notification | Post-commit handler | Should not roll back a successful business commit |

---

## 5. What the relay does

The relay is the background loop that turns durable outbox rows into RabbitMQ messages. It is the **retry-from-durable-record** process: something has to act on the staged intent with retries, because the request path cannot (broker may be down, process may die) and the consumer cannot (it only runs once the message arrives).

There is typically one relay per `DbContext` / bounded context, because each context owns its own outbox table. A bad tick is logged and does not kill the loop.

```text
every ~5 seconds:
  claim a batch of unprocessed rows
  publish each claimed row to RabbitMQ
  mark successfully published rows as processed
  delete old processed rows past retention
  measure backlog (e.g. oldest eligible age)
```

### Claim

Multiple pods may run the same relay, so a row is claimed with a **lease** (for example ~120 seconds), often via a multi-step conditional update:

```text
ProcessedAt IS NULL
AND FaultedAt IS NULL
AND (LockedUntil IS NULL OR LockedUntil < now)
ORDER BY OccurredAt, Id
```

Ordering by `OccurredAt` (not a random-sorted Guid `Id`) keeps "publish oldest first" meaningful within a table. The relay sets `LockedBy` / `LockedUntil`. If the pod dies, the lease expires and another relay can claim the row. Clock authority for lease and eligibility should be the database (e.g. SQL Server `SYSDATETIMEOFFSET()`), not individual app clocks.

A typical claim batch size is on the order of 100 rows per tick. The lease makes multi-replica relays **failover**, not systematic duplicators — though claim-loss still contributes to at-least-once delivery.

### Publish and mark

For each claimed row, the relay:

1. Resolves `MessageType` (unresolvable type → fault immediately)
2. Deserializes `Payload`
3. Calls `bus.Publish(...)` (with a publish timeout)
4. Updates `ProcessedAt` if publish succeeds and the lease is still held

If mark-processed affects zero rows (lease lost), another relay may republish — again, at-least-once.

Failure handling is different depending on the failure:

| Situation | Relay behavior |
|-----------|----------------|
| Broker down or publish timeout | Abort the batch; **do not** consume an attempt |
| One row has bad payload or another row-level error | Increment `Attempts`, set backoff via `LockedUntil` |
| Typical backoff | Exponential, e.g. `min(2^attempts × 30s, 1h)` |
| Row keeps failing | Park it with `FaultedAt` after a limit (e.g. 5 attempts) |
| Unresolvable message type | Fault immediately |
| Relay crashes after publish but before marking processed | Row is published again later |

The last case is why this pattern gives **at-least-once** delivery, not exactly-once delivery.

---

## 6. At-least-once means consumers must be idempotent

The outbox prevents silent loss after a successful commit, but it does not prevent duplicates.

This can happen:

```text
relay publishes to RabbitMQ
consumer receives the event
relay crashes before ProcessedAt is written
lease expires
another relay publishes the same row again
consumer receives the same event again
```

That duplicate is acceptable only if the consumer is idempotent.

Common strategies:

| Strategy | Example |
|----------|---------|
| Existing-state check | If the message is already processed, return |
| Unique business key | WhatsApp `wamid` unique index prevents duplicate inbound messages |
| `CorrelationId` dedupe | Skip work already recorded for this correlation (often with a unique index) |
| State machine guard | Ignore a stale `Delivered` event if the message is already terminal |
| Snapshot staleness guard | Event carries a timestamp/version; consumer skips if state is newer |

The mental model is: the relay guarantees "you will be told at least once"; the consumer guarantees "telling me twice will not corrupt state."

---

## 7. Worked example: inbound WhatsApp message

An inbound webhook is a good example because it crosses several contexts and shows the repeating unit three times.

```text
Meta webhook
  |
  v
TXN 1 - WhatsApp DB
  store raw WhatsappEnvelope
  stage outbox rows (same SaveChanges):
    - envelope to message processing
    - template status check
    - account status check
    - broadcast status check
  commit
  return 200 to Meta          ← nothing on RabbitMQ yet; delivery is now a DB guarantee
  |
  v
WhatsApp relay publishes the relevant event
  |
  v
Consumer parses the envelope
  envelope already processed? → return
  existing WAMIDs? → skip duplicates
  creates inbound WhatsappMessage rows
  |
  v
TXN 2 - WhatsApp DB
  store WhatsappMessage rows
  stage DispatchInboundWhatsappMessageCreated
  commit
  |
  v
WhatsApp relay publishes again
  |
  v
Consumer creates or finds chat session
  creates inbound SessionMessage
  |
  v
TXN 3 - Chat DB
  store SessionMessage
  stage DispatchInboundSessionMessageAdded
  commit
  |
  v
Chat relay publishes
  |
  v
Consumers push SignalR notifications and agent updates
```

When the webhook's `SaveChangesAsync` returns, one transaction has committed the envelope **and** the outbox rows. Fast ACK to Meta; the broker hop happens later.

The repeating unit is always the same:

1. Change state in the current context
2. Stage an outbox row in that same context and transaction (via a pre-commit transactional domain event handler)
3. Let that context's relay publish it
4. Make the next consumer idempotent

This design also explains why inbound delivery may have more latency than outbound delivery. A single outbound send may need one outbox hop. An inbound webhook can need three hops: envelope → message → chat.

---

## 8. Latency and ordering

If the relay polls every 5 seconds, the healthy-case delay is not a fixed 5 seconds. It is usually:

```text
0 to 5 seconds per outbox hop
average about 2.5 seconds per hop
```

The relay drains **everything claimable** each tick, then sleeps. The interval is a poll delay, not a per-message throttle: a thousand rows ready in one window publish together on the next tick.

The HTTP request does not wait for that delay. `SaveChangesAsync` returns immediately; a webhook can return `200` right away. The delay applies to the **downstream side effect**: sending to WhatsApp, notifying another service, pushing SignalR, and so on.

Rough multi-hop intuition:

| Path | Outbox hops | Healthy-case async delay (order of magnitude) |
|------|-------------|-----------------------------------------------|
| Outbound (agent → customer) | ~1 | up to ~5s before the side effect starts |
| Inbound (customer → agent screen) | ~3 | up to ~15s worst case, often lower on average |

Latency can grow when:

| Cause | Result |
|-------|--------|
| Broker outage | Rows wait until the broker recovers (attempts not burned on broker-down) |
| Backlog above single-worker throughput | Rows wait behind earlier eligible rows |
| Poison row | That row backs off or gets faulted; others continue |
| Relay deployment/crash | Rows wait until a relay is running again, then drain |

None of those lose the message — they only delay it.

**Trade-off vs. direct publish:** direct publish was nearly immediate but lossy; the outbox trades a few seconds of async latency for guaranteed delivery. If needed, operators can lower the poll interval or add a post-commit "wake" so the relay drains immediately instead of waiting for the next poll (wake is optional; plain polling is the simple default).

### Ordering

Ordering is intentionally limited.

- **Cross-context:** non-goal. Independent relays, no shared sequence, no ordered-delivery primitive across tables.
- **`OccurredAt`:** staging time on the producing app server — a within-context "oldest first" claim hint, not a global clock. Clock skew makes cross-context comparison meaningless.
- **Even within one context:** the relay may claim in `OccurredAt` order per tick, but the bus delivers concurrently, and backoff/redelivery reorder. Any event may arrive late, out of order, or twice.

What replaces strict ordering — **order-tolerant consumers**:

1. Idempotency / dedupe (state checks, `CorrelationId` + unique index, business keys).
2. State-based convergence — act on *current committed state*, not on arrival order; a stale event finds state already advanced and skips.
3. Snapshot staleness guards — event carries a snapshot; consumer skips if superseded.

The only ordering that really holds is **causal**, from pipeline shape: event B cannot exist until event A was consumed and produced a state change. Independent events have no defined order, and no consumer should assume one.

---

## 9. Why a relay is required

Sometimes the tempting alternative is:

> "After committing the outbox row, why not publish immediately from the request thread?"

That is still only one fragile attempt with no retry loop:

```text
COMMIT ✅ → Publish() → broker down / timeout / pod dies → event stuck, nobody retries
```

The consumer cannot fix hop A either. Hop B (broker → consumer) is already reliable; hop A (app → broker) is the fragile part. The consumer only runs if the message actually arrives.

| Property | Direct / inline publish after commit | Outbox relay |
|----------|--------------------------------------|--------------|
| Retry after outage or crash | No — one shot | Yes — from the durable row |
| Request succeeds while broker is down | No — fails or loses the event | Yes — commit succeeds, relay catches up |
| Publishes only committed state | Depends on ordering (phantoms possible) | Yes — only committed rows |
| Poison message off the request path | No | Yes — backoff / fault-park |
| Survives process crash after commit | No | Yes |

An "optimistic publish + relay sweeper" design is possible for lower latency, but you still need the relay (for failures) and idempotent consumers (both paths can deliver → duplicates). Relay-only is simpler and usually a good starting point; the cost is that ~0–5s poll delay per hop.

**One-liner:** the outbox row records the *intent*; something must *act* on it with retries from that durable record. The request path cannot own that responsibility; a separate retrying process (the relay) can.

---

## 10. What is guaranteed

| Question | Answer |
|----------|--------|
| Can an event be lost after the business commit? | Not if the outbox row was committed and relays keep running |
| Can a rolled-back business write publish an event? | No — the outbox row rolls back too |
| Is delivery exactly once? | No — it is **at-least-once** |
| Who handles duplicates? | Consumers (must be idempotent) |
| Is ordering guaranteed across contexts? | No |
| Is ordering strict within one context? | No — oldest-first claim only; bus and retries reorder |
| Is the publish synchronous with the API request? | No — the relay publishes asynchronously |
| Does the row disappear immediately after publish? | No — marked processed, then cleaned up after retention |
| Are faulted rows auto-deleted? | No — operators handle poison rows |

The trade-off is straightforward:

```text
direct publish: lower latency, possible silent loss
outbox relay: small async delay, durable retry
```

For user-visible or money-related workflows, silent loss is usually the worse failure.

---

## 11. Final mental model

The database transaction is the commit point for two facts:

1. The business state changed.
2. The system owes the outside world an integration event.

The relay owns the risky network hop from database to broker. It retries from a durable row until the event is published or explicitly faulted.

The consumer owns duplicate safety.

```text
SQL transaction
  business row + outbox row     ← atomic intent
        |
        v
relay
  claim + publish + mark processed   ← at-least-once pump
        |
        v
RabbitMQ
        |
        v
idempotent consumer             ← absorbs duplicates / reorder
```

You cannot make SQL Server and RabbitMQ share one ACID transaction. The transactional outbox pattern avoids needing that. It makes the state change and the publish intent atomic, then lets a retrying background process finish the cross-system work.
