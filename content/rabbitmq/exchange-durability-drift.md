+++
title = "First Declarer Wins — Debugging RabbitMQ Exchange State Drift"
date = 2026-06-22

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ", "MassTransit", "debugging", "messaging"]
+++

In production, the entire WeChat inbound message path stopped working. I searched through the code, but there was not a single line to change. The culprit was a **wrong exchange definition left behind in the broker**, and the reason it could happen was one key property of RabbitMQ.

## Symptom

The following error was repeating endlessly in production.

```text
MassTransit.RabbitMqConnectionException: ReceiveTransport faulted ...
RabbitMQ.Client.Exceptions.OperationInterruptedException:
  code=406, text='PRECONDITION_FAILED - inequivalent arg 'durable'
  for exchange '...:DispatchInboundWechatMessageCreated'
  in vhost '/': received 'true' but current is 'false''
```

Interpreted, it means this:

- Our app is trying to declare the exchange with **`durable=true`** (`received 'true'`).
- An exchange with the same name **already exists** in the broker with **`durable=false`** (`current is 'false'`).
- RabbitMQ says, "Those do not match. I reject this." → `406 PRECONDITION_FAILED` → channel closed.

As a result, MassTransit's receive transport entered a fault → reconnect → fault again loop, and **WeChat inbound message processing stopped completely.**

## Intuition: what kind of situation is this?

Imagine an electrical outlet, the exchange, already installed in a wall. The first person who installs that outlet **sets the specification and nails it in place.** Once installed, the specification cannot be changed.

- Our app expects a **220V specification, durable**, and tries to plug in.
- Someone previously installed a **110V specification, non-durable**, outlet in that spot.
- The plug does not fit. RabbitMQ does not convert the outlet to 220V. It simply says, "Specification mismatch," and closes the channel.

Two key intuitions:

1. **The first declarer determines the exchange specification, and it is immutable after that.** → first-declarer-wins.
2. **If the specification does not match, RabbitMQ rejects it instead of fixing it.** → `PRECONDITION_FAILED`.

This is not solved by "fixing the code." The app can recreate the exchange with the correct specification only after **the wrong exchange left in the broker is removed**.

## RabbitMQ properties that caused this

### Declaration idempotency + equivalence check

`exchange.declare` is **idempotent**. Declaring the same thing many times is fine, but only when the arguments are identical. If you declare an existing exchange with **different arguments**, RabbitMQ rejects it instead of modifying it.

> "The arguments differ from the existing one. I will not do that." → `406 PRECONDITION_FAILED`, channel closed.

Exchange properties such as `type`, `durable`, `auto_delete`, `internal`, and `arguments` are **fixed at creation time and cannot be changed afterward.** To change them, you must delete the exchange and create it again.

### Durable vs transient

| | Meaning |
|---|---|
| `durable=true` | The exchange definition is stored on disk → **survives broker restarts** |
| `durable=false` | Exists only in memory → disappears when the broker restarts |

The problematic exchange had `durable=false`, while our app wanted `durable=true`.

### Auto-delete

When `auto-delete=true`, the exchange deletes itself when the last binding disappears. In the management UI, this appears as `AD` in the Features column. The problematic exchange had the combination `AD` + non-durable, which is the typical shape of a **temporary exchange**.

### Topology persistence

Exchanges, queues, and bindings are **server-side broker state**. Even after the client connection that created them is gone, the definitions remain in the broker. Because the broker container lifetime was `Persistent`, the wrong definition created once kept surviving.

## Why this was not a code bug

At first, I suspected that "somewhere in the code, something must be declaring it with `durable=false`." Two pieces of evidence broke that hypothesis.

### Evidence 1: there was no durability setting anywhere in the full git history

```bash
git log --all -S "Durable" -- '*.cs'                                      # → none
git log --all -G "Durable|AutoDelete|Temporary|PublishTopology" -- '*.cs'  # → none
```

There had never been a commit that set or changed durability/auto-delete. The code had always used MassTransit's default, which is durable.

### Evidence 2: only WeChat was broken, while WhatsApp/LINE were fine

The inbound dispatch code for the four contexts, WeChat, WhatsApp, LINE, and Internal, is **structurally identical**.

> If this were a code/configuration problem, all four should have broken **the same way**. But only WeChat was broken.

That asymmetry definitively ruled out code as the cause. **This was neither a code bug nor a design flaw. It was a state mismatch between the wrong exchange definition left in the broker and the declaration made by the code.**

## Why only WeChat? first-declarer-wins applies per exchange

The key is that durability is fixed **per exchange**.

- WhatsApp/LINE/Internal exchanges: our app was the first declarer, durable → fixed as durable → no problem.
- WeChat exchange: **someone else first declared it as transient, non-durable + auto-delete** → fixed as AD → every later durable declaration from our app conflicted.

It is like installing four outlets in a wall: three were installed according to the specification, but one was installed first by someone else with the wrong specification. The blueprint, the code, is identical for all four.

> **WhatsApp and LINE were not "safe"; they were "lucky."** If a wrong client first declares their exchanges as transient, they can break the same way.

## Diagnosis: is the culprit still alive?

Before deleting the exchange, I needed one check because there were two possible scenarios.

- **(A) Live external publisher**: it is still recreating this exchange as transient → even if I delete it, it immediately comes back as AD. That client must be stopped first.
- **(B) Old orphaned leftover**: someone created it once in the past, and that client is already gone → deleting it once is enough.

I checked the broker's connection list.

```bash
rabbitmqctl list_connections name peer_host user channels
```

There were **exactly two connections**, both from our app: 59 channels, meaning MassTransit's full bus and two replicas. No suspicious third-party connection existed → **scenario (B), old leftover.** It was safe to delete.

## Fix: delete it and let the app recreate it

```bash
rabbitmqadmin -u "$RABBITMQ_DEFAULT_USER" -p "$RABBITMQ_DEFAULT_PASS" --vhost / \
  delete exchange --name '...:DispatchInboundWechatMessageCreated'
```

Immediately after deletion, MassTransit's reconnect loop declared the exchange again on the next attempt, this time as **durable**. There was no conflicting existing exchange anymore, so it succeeded.

```bash
rabbitmqctl list_exchanges name durable auto_delete | grep DispatchInboundWechat
# → ... true  false   (durable=true, auto_delete=false) ✅
```

In the management UI, Features returned to `D`. The `precondition_failed` logs stopped, and the `ReceiveTransport faulted ... Retrying` loop stopped as well.

Deleting an exchange **does not touch queues or the messages inside them.** Only the exchange and bindings disappear, and MassTransit automatically recreates the bindings too.

## Preventing recurrence

This is the kind of problem code review cannot catch, because the problem is broker state. A **runtime monitoring guard** was needed.

I added `RabbitMqExchangeTopologyGuard`.

- A read-only `BackgroundService` that runs at startup and every 10 minutes
- Calls the RabbitMQ management API, `/api/exchanges/{vhost}`, and logs a `LogWarning` when an `Omni.`-prefixed exchange is **non-durable or auto-delete**
- Excludes legitimate temporary endpoints such as `bus-*` and `amq.*`, because MassTransit's temporary bus/reply endpoints are supposed to be AD
- **Does not mutate the broker.** Actual recovery is still handled manually by delete → redeclare

This guard **detects** the issue, but does not **prevent or auto-recover** it. To really reduce recurrence, the next steps are:

1. Promote `LogWarning` to a **real alert or health check**, because logs alone can be buried.
2. When drift occurs, also query `/api/connections` and log the suspected culprit connection.
3. **Audit environments**: check whether any non-prod instance has the production broker connection string.

## Lessons

1. **If it is not in the code, it may not be a code problem.** First use git pickaxe (`-S`/`-G`) to check whether any commit ever touched that setting. It saves time otherwise spent digging in the wrong place.

2. **A distributed-system bug can live in "state," not "code."** Long-lived external state such as brokers, databases, and caches has its own truth separate from the code.

3. **Asymmetry is a strong diagnostic signal.** "The code is identical, so why only one?" usually means the cause is not symmetric code/configuration, but instance-specific state.

4. **Remember first-declarer-wins.** RabbitMQ exchange/queue arguments are fixed and immutable at first declaration. When several clients share the same broker, whoever declares first can create permanent differences.

5. **"Not broken" is different from "safe."** WhatsApp/LINE were not structurally protected; they were lucky.

---

## Three-line summary

1. RabbitMQ exchange properties such as durable and auto-delete are **fixed at first declaration and immutable afterward**. Redeclaring with different arguments does not update it; it closes the channel with `406 PRECONDITION_FAILED` instead. This is first-declarer-wins.
2. Only the WeChat exchange broke not because of code, but because of **state mismatch**. A transient exchange previously declared by someone remained in the broker and blocked every durable declaration from our app.
3. The fix was just deleting the wrong exchange. After first confirming with `list_connections` that no live redeclarer existed, deleting it let the app recreate it as durable automatically.
