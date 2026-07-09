+++
title = "Transactional Outbox Pattern: DB와 브로커를 일관되게 맞추기"
date = 2026-07-09

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ", "MassTransit", "outbox", "messaging", "EF Core", "distributed systems"]
+++

Transactional outbox pattern은 막으려는 실패를 먼저 보면 이해하기 쉽다.

채팅 시스템에서 상담원이 고객에게 답장을 보낸다. API는 답장을 SQL Server에 저장하고 `200 OK`를 반환한다. 상담원 대시보드에도 메시지가 보인다. 그런데 고객은 답장을 받지 못한다. 프로세스가 RabbitMQ 이벤트를 발행하기 전에 죽었고, 그 이벤트가 없어서 WhatsApp을 호출할 컨슈머도 실행되지 않았기 때문이다.

DB에는 답장이 있다. RabbitMQ에는 메시지가 없다. 컨슈머는 실행되지 않는다. 에러도 없고, 알림도 없다.

이게 dual-write problem이다. 하나의 요청이 DB와 브로커라는 독립적인 두 시스템을 함께 바꾸려고 하지만, 두 시스템은 하나의 트랜잭션을 공유하지 않는다. 규모가 커지면 "거의 항상 된다"(예: 99.9%)도 조용한 데이터 무결성 버그다. 사용자에게 보이거나 돈과 관련된 이벤트에는 허용하기 어렵다.

---

## 1. 안전하지 않은 버전

처음 구현은 보통 이렇게 생긴다.

```csharp
session.AddOutboundMessage(text);
await dbContext.SaveChangesAsync();
await publishEndpoint.Publish(new MessageCreated(...));
```

happy path에서는 괜찮아 보이지만, 두 줄 사이에 위험한 틈이 있다.

```text
SaveChangesAsync()가 SQL에 커밋
    |
    |  프로세스 크래시, 브로커 장애, 타임아웃, 파드 재시작
    v
Publish(...)가 RabbitMQ에 도달하지 못함
```

그 뒤의 상태는 이렇다.

| 시스템 | 상태 |
|--------|------|
| SQL Server | 비즈니스 행이 존재 |
| RabbitMQ | 통합 이벤트 없음 |
| Consumer | 실행되지 않음 |

이게 **lost event**다.

순서를 뒤집어도 해결되지 않는다.

```csharp
await publishEndpoint.Publish(new MessageCreated(...));
await dbContext.SaveChangesAsync();
```

이번에는 DB 변경이 나중에 롤백됐는데도 컨슈머가 이벤트를 보고 행동할 수 있다. 이게 **phantom event**다.

| 순서 | 실패 구간 | 결과 |
|------|-----------|------|
| DB 먼저, 그다음 publish | 커밋 성공, publish 실패 | Lost event |
| Publish 먼저, 그다음 DB | Publish 성공, 커밋 실패 | Phantom event |

문제는 순서가 아니다. 독립적인 두 시스템을 하나의 atomic write처럼 다루려는 것이 문제다.

### 두 hop, 하나만 취약하다

경로를 나누면 이해가 쉽다.

```text
[app이 SQL에 커밋] ── hop A ──▶ [RabbitMQ에 메시지] ── hop B ──▶ [consumer 실행]
```

- **Hop B (broker → consumer)** 는 실무에서 이미 충분히 신뢰할 수 있다. MassTransit가 retry, redelivery, DLQ를 한다. Hop B에는 outbox가 필요 없다.
- **Hop A (app → broker)** 에는 트랜잭션 보장이 없다. `Publish`는 DB 트랜잭션의 일부가 아니라, 별도 시스템으로의 네트워크 호출이다. 이벤트가 조용히 사라지는 지점이 여기다.

그래서 "integration handler가 가끔 안 돈다"고 말할 때, 메시지가 브로커에 **도달했다면** handler는 사실상 100% 돈다. 진짜 버그는 메시지가 브로커에 아예 안 도착하는 경우다.

Hop A의 전형적인 실패 모드 (모두 DB는 커밋됐고 bus에는 아무것도 없음):

1. Broker down / unreachable — `Publish`가 throw.
2. Publish timeout 또는 호출 중 connection drop.
3. 커밋 직후, publish 전에 프로세스 종료 (pod eviction, deploy, OOM).
4. 로컬에서는 publish "성공"처럼 보이지만 메시지가 durable하게 안 남음 (client buffer 유실, flush 전 broker crash).

---

## 2. Outbox 아이디어

요청 안에서 RabbitMQ에 바로 publish하지 않는다. 대신 비즈니스 변경과 "이 이벤트를 나중에 발행해야 한다"는 기록을 **같은 DB 트랜잭션**에 쓴다.

```sql
BEGIN TRANSACTION;

INSERT INTO SessionMessages (...);

INSERT INTO OutboxMessages
    (Id, MessageType, Payload, CorrelationId, OccurredAt, ProcessedAt)
VALUES
    (..., 'DispatchMessageCreated', '{...}', ..., SYSUTCDATETIME(), NULL);

COMMIT;
```

커밋 결과는 두 가지뿐이다.

| 커밋 결과 | 비즈니스 행 | Outbox 행 |
|-----------|-------------|-----------|
| 성공 | 있음 | 있음 |
| 실패 / 롤백 | 없음 | 없음 |

메시지는 저장됐는데 발행해야 한다는 기록만 없는 상태가 없다.

### DB도 외부 시스템인데, 왜 outbox DB write는 "안전"한가?

Outbox write는 **독립된 두 번째 write가 아니다**. 비즈니스 변경과 *같은* 트랜잭션의 일부다. partial failure를 all-or-nothing으로 바꾼다.

| 패턴 | Write | 한쪽만 실패할 때 middle state |
|------|-------|------------------------------|
| Direct publish | SQL + RabbitMQ (두 시스템) | 있음 — 상태 저장, 이벤트 유실 (또는 phantom) |
| Outbox | SQL만 (business row + outbox row) | 없음 — 둘 다 있거나 둘 다 rollback |

커밋이 실패하면 비즈니스 행도 저장되지 않았다. command는 에러를 반환하고, caller가 재시도하며, 절반만 일어난 상태가 없다. 커밋 중 crash도 SQL Server WAL 아래에서는 안전하다. 트랜잭션은 전부 적용되거나 전부 버려진다.

커밋 시점에 RabbitMQ는 아직 업데이트하지 않는다. 의도적인 선택이다. 커밋 후에는 백그라운드 **relay**가 outbox 테이블을 읽고 재시도와 함께 발행한다. 그 경로의 실패는 **유실이 아니라 지연**이다. publish 실패 → row는 unprocessed로 남음 → 재시도; publish 후 "mark processed" 전 crash → 다시 publish (중복) → consumer가 dedupe.

Outbox 행은 "프로세스가 publish할 때까지 살아 있기를 바란다"를 "DB가 이 이벤트를 반드시 발행해야 한다는 사실을 내구성 있게 기록한다"로 바꾼다.

여러 context (`DbContext` / bounded context)가 있는 시스템에서는 EF Core 기본 single-context outbox가 잘 안 맞는 경우가 많다. 보통 context마다 자신만의 `OutboxMessage` 테이블과 relay를 둔다.

---

## 3. Outbox 행에 들어가는 것

`OutboxMessage` 행은 내구성 있는 publish 작업이다.

| 컬럼 | 목적 |
|------|------|
| `Id` | 행 식별자 |
| `MessageType` | Relay가 payload를 역직렬화할 때 쓰는 CLR 타입 이름 |
| `Payload` | 직렬화된 integration event |
| `CorrelationId` | hop 사이의 관련 작업 연결 및 dedupe |
| `OccurredAt` | 이벤트가 stage된 시각 (claim 순서에 사용) |
| `ProcessedAt` | relay가 성공적으로 publish하기 전까지 `NULL` |
| `Attempts` | row-level 실패 횟수 |
| `FaultedAt` | poison row로 park될 때 설정 |
| `LockedBy`, `LockedAt`, `LockedUntil` | 여러 relay instance가 조율하기 위한 lease 필드 |

Relay가 publish하기 전:

```text
Id | MessageType              | ProcessedAt | Attempts
1  | DispatchMessageCreated   | NULL        | 0
```

성공적으로 publish한 뒤:

```text
Id | MessageType              | ProcessedAt                | Attempts
1  | DispatchMessageCreated   | 2026-07-09 10:00:07 +00:00 | 0
```

### Mark-then-sweep (publish 직후 삭제하지 않음)

성공해도 relay는 행을 바로 지우지 않는다. 표시만 한다.

```text
staged       ProcessedAt = NULL, FaultedAt = NULL   ← claim 가능
    │  relay publish 성공
    v
processed    ProcessedAt = <timestamp>              ← 남음, 더 이상 claim 안 됨
    │  약 7일 후
    v
deleted      retention pass가 삭제
```

| 상태 | `ProcessedAt` | `FaultedAt` | 테이블에? | 삭제? |
|------|:-------------:|:-----------:|:---------:|-------|
| Pending | NULL | NULL | 예 | — |
| Published OK | 설정됨 | NULL | 예 | retention 후 (예: 7일) |
| Fault-parked | NULL | 설정됨 | 예 | **자동 삭제 없음** (운영자만) |

Delete-on-publish 대신 mark-then-sweep을 쓰는 이유:

- Publish 경로를 단일 컬럼 update로 가볍게 유지.
- Hot path에서 lock/index churn을 줄이도록 삭제는 batch.
- 짧은 audit window로 "이게 발행됐나, 언제?"에 답할 수 있음.

Faulted row는 의도적으로 운영자가 볼 때까지 남긴다. Poison message를 자동으로 지우면 문제가 조용히 사라진다. (Payload 데이터도 남으므로 운영·개인정보 검토에서 감안해야 한다.)

---

## 4. 행은 어디서 만들어지나

내가 살펴본 구현에서는 command handler가 모든 위치에서 직접 outbox row를 넣지 않는다. 흐름은 domain event와 EF Core `SaveChanges`를 중심으로 잡혀 있다.

"이전" 형태는 **post-commit** domain event handler(또는 command 코드)가 `IPublishEndpoint`를 주입받아 직접 publish하는 것이었다. "발행 의도"의 durable 기록이 없다. "이후" 형태는 같은 트랜잭션에 row를 stage한다.

```csharp
// 이전: 커밋 후 직접 publish — gap에서 프로세스가 죽으면 유실
await publishEndpoint.Publish(new DispatchInboundSessionMessageAdded(...));

// 이후: 같은 SaveChanges 트랜잭션에 stage; relay가 나중에 publish
integrationEventOutbox.Add(new DispatchInboundSessionMessageAdded(...));
```

```text
Command handler
  |
  | aggregate mutation이 메모리에서 domain event를 발생시킴
  v
SaveChangesAsync()
  |
  | PRE-COMMIT: DomainEventPublishInterceptor가 transactional handler 실행
  |   |
  |   v
  | ITransactionalDomainEventHandler<T>
  |   |
  |   v
  | integrationEventOutbox.Add(integrationEvent)
  |
  | EF가 business row + OutboxMessage row를 INSERT
  v
COMMIT
  |
  v
POST-COMMIT handler: SignalR, cache refresh, 기타 non-transactional side effect
```

중요한 점은 `integrationEventOutbox.Add(...)`가 아무것도 publish하지 않는다는 것이다. 그냥 EF가 추적하는 insert다.

```csharp
public void Add(IIntegrationEvent integrationEvent)
    => dbContext
        .Set<OutboxMessage>()
        .Add(OutboxMessage.Create(integrationEvent));
```

Outbox message의 "처리"는 두 시점으로 나뉜다. 어느 쪽도 "integration-event publish interceptor"가 아니다.

| 단계 | 언제 | 무엇에 의해 |
|------|------|-------------|
| **Persisted** | 동기, 트리거가 된 `SaveChangesAsync` 커밋 시 | `Add` 이후 일반 EF change tracking |
| **Published** | 비동기, 다음 relay tick (보통 0–5초) | `OutboxRelayService` 백그라운드 poller |

`IntegrationEventPublishInterceptor`는 설계상 없다. Domain event는 인터셉터(`DomainEventPublishInterceptor`)가 *dispatch*하고, integration event는 그렇지 않다. 커밋 시점에 publish하는 인터셉터를 두면 outbox가 없애려던 crash-in-the-gap 문제를 다시 만든다.

Staging을 안전하게 유지하는 규칙:

- Transactional handler는 스스로 `SaveChangesAsync`를 **호출하지 않는다**. `Add`만 하고 트리거 save에 실려 간다.
- Transactional handler가 throw하면 전체 save가 실패한다. 비즈니스 상태와 publish intent는 함께 성공하거나 함께 실패한다.
- 다른 context가 반드시 받아야 하는 integration event → pre-commit transactional handler.
- SignalR / cache / UI → post-commit handler (성공한 비즈니스 커밋을 되돌리면 안 됨).

| 일 | 단계 | 이유 |
|----|------|------|
| 다른 context가 반드시 받아야 하는 integration event | Pre-commit transactional handler | 비즈니스 상태와 atomic하게 커밋돼야 함 |
| SignalR push, cache refresh, UI notification | Post-commit handler | 성공한 비즈니스 커밋을 되돌리면 안 됨 |

---

## 5. Relay가 하는 일

Relay는 내구성 있는 outbox row를 RabbitMQ 메시지로 바꾸는 백그라운드 loop다. **durable record에서 시작하는 retry loop**다. stage된 intent를 재시도와 함께 처리하는 무언가가 필요하다. request path는 그 책임을 질 수 없고 (broker down, process die), consumer도 질 수 없다 (메시지가 도착해야만 실행된다).

보통 `DbContext` / bounded context마다 outbox table이 있으므로 relay도 context마다 하나씩 둔다. 한 tick이 실패해도 로그를 남기고 loop는 죽지 않는다.

```text
약 5초마다:
  처리되지 않은 row batch claim
  claim한 row를 RabbitMQ에 publish
  성공한 row를 processed로 표시
  retention을 지난 오래된 processed row 삭제
  backlog 측정 (예: oldest eligible age)
```

### Claim

여러 pod가 같은 relay를 실행할 수 있으므로 row는 **lease**(예: 약 120초)로 claim한다. 보통 multi-step conditional update 형태다.

```text
ProcessedAt IS NULL
AND FaultedAt IS NULL
AND (LockedUntil IS NULL OR LockedUntil < now)
ORDER BY OccurredAt, Id
```

`OccurredAt` 순서로 claim한다 (SQL Server에서 임의 정렬되는 Guid `Id`가 아님). 같은 테이블 안에서는 "오래된 것부터 publish"가 의미가 있다. Relay는 `LockedBy` / `LockedUntil`을 설정한다. Pod가 중간에 죽으면 lease가 만료되고 다른 relay가 그 row를 다시 claim할 수 있다. Lease·eligibility의 clock authority는 앱 서버가 아니라 DB(예: SQL Server `SYSDATETIMEOFFSET()`)여야 한다.

전형적인 claim batch 크기는 tick당 100 row 정도다. Lease 덕분에 multi-replica relay는 **체계적 중복 발행기가 아니라 failover**다. 다만 claim-loss는 여전히 at-least-once에 기여한다.

### Publish and mark

Claim한 row마다 relay는 다음을 수행한다.

1. `MessageType` resolve (해석 불가면 즉시 fault)
2. `Payload` deserialize
3. `bus.Publish(...)` 호출 (publish timeout 포함)
4. 성공하고 lease를 아직 들고 있으면 `ProcessedAt` update

Mark-processed의 affected row가 0이면 (lease 상실) 다른 relay가 다시 publish할 수 있다. 역시 at-least-once.

실패 처리 방식은 실패 종류에 따라 다르다.

| 상황 | Relay 동작 |
|------|------------|
| Broker down 또는 publish timeout | Batch 중단, attempt를 **소비하지 않음** |
| 특정 row의 잘못된 payload 등 row-level error | `Attempts` 증가, `LockedUntil`으로 backoff |
| 전형적인 backoff | Exponential, 예: `min(2^attempts × 30s, 1h)` |
| 계속 실패하는 row | 한도(예: 5 attempts) 이후 `FaultedAt`으로 park |
| 해석 불가 message type | 즉시 fault |
| Publish 성공 후 processed 표시 전에 relay crash | 나중에 같은 row가 다시 publish됨 |

마지막 경우 때문에 이 패턴은 exactly-once가 아니라 **at-least-once** delivery를 제공한다.

---

## 6. At-least-once는 consumer idempotency가 필요하다는 뜻

Outbox는 성공한 커밋 이후의 조용한 유실을 막지만, 중복을 막지는 않는다.

이런 일이 가능하다.

```text
relay가 RabbitMQ에 publish 성공
consumer가 이벤트 수신
relay가 ProcessedAt을 쓰기 전에 crash
lease 만료
다른 relay가 같은 row를 다시 publish
consumer가 같은 이벤트를 다시 수신
```

이 중복은 consumer가 idempotent할 때만 안전하다.

자주 쓰는 방법:

| 전략 | 예 |
|------|----|
| Existing-state check | 메시지가 이미 처리됐으면 return |
| Unique business key | WhatsApp `wamid` unique index로 중복 inbound message 방지 |
| `CorrelationId` dedupe | 이미 기록된 correlation이면 skip (보통 unique index와 함께) |
| State machine guard | 메시지가 이미 terminal 상태면 늦은 `Delivered` 이벤트 무시 |
| Snapshot staleness guard | 이벤트의 timestamp/version보다 현재 상태가 더 최신이면 skip |

Mental model은 이렇다. Relay는 "적어도 한 번은 알려준다"를 보장하고, consumer는 "두 번 알려줘도 상태가 망가지지 않는다"를 보장한다.

---

## 7. 예시: Inbound WhatsApp message

Inbound webhook은 여러 context를 건너고, 같은 반복 단위가 세 번 나타나기 때문에 좋은 예시다.

```text
Meta webhook
  |
  v
TXN 1 - WhatsApp DB
  raw WhatsappEnvelope 저장
  같은 SaveChanges에서 outbox row stage:
    - envelope to message processing
    - template status check
    - account status check
    - broadcast status check
  commit
  Meta에 200 반환          ← 아직 RabbitMQ에는 없음; 이제 delivery는 DB 보장
  |
  v
WhatsApp relay가 관련 event publish
  |
  v
Consumer가 envelope parse
  envelope already processed? → return
  existing WAMID? → 중복 skip
  inbound WhatsappMessage row 생성
  |
  v
TXN 2 - WhatsApp DB
  WhatsappMessage row 저장
  DispatchInboundWhatsappMessageCreated stage
  commit
  |
  v
WhatsApp relay가 다시 publish
  |
  v
Consumer가 chat session 생성 또는 조회
  inbound SessionMessage 생성
  |
  v
TXN 3 - Chat DB
  SessionMessage 저장
  DispatchInboundSessionMessageAdded stage
  commit
  |
  v
Chat relay publish
  |
  v
Consumer가 SignalR notification과 agent update 처리
```

Webhook의 `SaveChangesAsync`가 반환될 때, 한 트랜잭션이 envelope **와** outbox row를 함께 커밋했다. Meta에는 빠르게 200 ACK를 주고, broker hop은 나중에 일어난다.

반복 단위는 항상 같다.

1. 현재 context에서 상태 변경
2. 같은 context, 같은 transaction에 outbox row stage (pre-commit transactional domain event handler)
3. 그 context의 relay가 publish
4. 다음 consumer는 idempotent하게 처리

이 구조 때문에 inbound delivery가 outbound delivery보다 더 늦을 수 있다. Outbound send는 outbox hop이 하나면 충분할 수 있지만, inbound webhook은 envelope → message → chat으로 3 hop을 거칠 수 있다.

---

## 8. Latency와 ordering

Relay가 5초마다 polling한다면 healthy case delay는 고정 5초가 아니다.

```text
outbox hop 하나당 0~5초
평균은 대략 2.5초
```

Relay는 매 tick마다 claim 가능한 것을 **전부** 비우고 잔다. Interval은 per-message throttle이 아니라 poll delay다. 한 window에 준비된 1000 row는 다음 tick에 함께 publish된다.

HTTP 요청은 이 delay를 기다리지 않는다. `SaveChangesAsync`는 바로 반환되고, webhook도 즉시 `200`을 줄 수 있다. Delay는 **downstream side effect**에 적용된다. WhatsApp 전송, 다른 서비스 알림, SignalR push 같은 것들이다.

다중 hop 대략 감:

| 경로 | Outbox hop | Healthy-case async delay (대략) |
|------|------------|----------------------------------|
| Outbound (상담원 → 고객) | ~1 | side effect 시작 전 최대 ~5초 |
| Inbound (고객 → 상담원 화면) | ~3 | worst ~15초, 평균은 보통 더 낮음 |

Latency가 커지는 경우:

| 원인 | 결과 |
|------|------|
| Broker outage | Broker가 회복될 때까지 row 대기 (broker-down에서 attempt 소모 안 함) |
| Single-worker 처리량 이상의 backlog | 먼저 eligible한 row 뒤에서 대기 |
| Poison row | 해당 row만 backoff 또는 fault; 나머지는 계속 |
| Relay deploy/crash | Relay가 다시 돌 때까지 대기 후 drain |

어느 경우도 메시지를 유실하지 않는다. 지연만 된다.

**Direct publish와의 trade-off:** direct publish는 거의 즉시지만 lossy했다. Outbox는 몇 초의 async latency를 대가로 보장된 delivery를 산다. 필요하면 poll interval을 줄이거나, post-commit "wake"로 다음 poll을 기다리지 않고 바로 drain할 수 있다 (wake는 선택; 단순 기본값은 plain polling).

### Ordering

Ordering도 의도적으로 제한적이다.

- **Cross-context:** non-goal. 독립 relay, 공유 sequence 없음, 테이블 간 ordered-delivery primitive 없음.
- **`OccurredAt`:** producing app server의 staging 시각 — context 안 "오래된 것부터" claim hint일 뿐 global clock이 아님. clock skew 때문에 cross-context 비교는 의미가 없다.
- **같은 context 안에서도:** tick마다 `OccurredAt` 순으로 claim할 수 있지만, bus는 concurrent delivery하고 backoff/redelivery가 순서를 바꾼다. 어떤 이벤트든 늦게, 순서 없이, 또는 두 번 올 수 있다.

Strict ordering을 대체하는 것은 **order-tolerant consumer**다.

1. Idempotency / dedupe (state check, `CorrelationId` + unique index, business key).
2. State-based convergence — 도착 순서가 아니라 *현재 committed state*를 보고 행동. 늦은 이벤트는 이미 앞선 상태를 보고 skip.
3. Snapshot staleness guard — 이벤트가 snapshot을 들고 오고, 이미 더 최신이면 skip.

실제로 성립하는 유일한 ordering은 파이프라인 모양에서 오는 **causal** order다. Event B는 event A가 소비되어 상태 변경을 만든 뒤에야 존재할 수 있다. 독립 이벤트에는 정의된 순서가 없고, consumer도 그걸 가정하면 안 된다.

---

## 9. 왜 relay가 필요한가

가끔 이런 생각이 든다.

> "Outbox row를 커밋한 뒤 request thread에서 바로 publish하면 안 되나?"

그건 여전히 retry loop 없는 fragile한 1회 시도다.

```text
COMMIT ✅ → Publish() → broker down / timeout / pod dies → event stuck, 아무도 재시도 안 함
```

Consumer도 hop A를 고칠 수 없다. Hop B (broker → consumer)는 이미 신뢰할 만하고, hop A (app → broker)가 취약한 부분이다. Consumer는 메시지가 실제로 도착해야만 돈다.

| 속성 | Commit 후 direct / inline publish | Outbox relay |
|------|-----------------------------------|--------------|
| Outage/crash 후 retry | 아니오 — 한 번뿐 | 예 — durable row에서 |
| Broker down 중에도 요청 성공 | 아니오 — 실패하거나 이벤트 유실 | 예 — 커밋 성공, relay가 catch-up |
| 커밋된 상태만 publish | 순서에 따라 다름 (phantom 가능) | 예 — 커밋된 row만 |
| Poison message를 request path 밖 처리 | 아니오 | 예 — backoff / fault-park |
| Commit 후 process crash 생존 | 아니오 | 예 |

Latency를 줄이기 위해 "optimistic publish + relay sweeper"도 가능하다. 하지만 그래도 relay는 필요하고 (실패 대비), consumer는 idempotent해야 한다 (두 path가 모두 deliver → 중복). Relay-only가 더 단순하고 보통 좋은 출발점이다. 대가는 hop당 ~0–5초 poll delay다.

**한 줄:** outbox row는 *intent*를 기록한다. 그 durable record에서 재시도와 함께 *행동*하는 무언가가 필요하다. Request path는 그 책임을 질 수 없고, 별도의 retry process(relay)가 질 수 있다.

---

## 10. 보장하는 것

| 질문 | 답 |
|------|----|
| 비즈니스 커밋 후 이벤트가 유실될 수 있나? | Outbox row가 함께 커밋되고 relay가 계속 돌면 아니다 |
| 롤백된 비즈니스 write가 이벤트를 publish할 수 있나? | 아니다. Outbox row도 함께 rollback된다 |
| Exactly-once delivery인가? | 아니다. **At-least-once**다 |
| 중복은 누가 처리하나? | Consumer (idempotent해야 함) |
| Context 사이 ordering이 보장되나? | 아니다 |
| 한 context 안에서 strict ordering인가? | 아니다 — oldest-first claim일 뿐; bus와 retry가 순서를 바꿈 |
| API 요청과 publish가 동기인가? | 아니다. Relay가 비동기로 publish한다 |
| Publish 후 row가 바로 사라지나? | 아니다. processed 표시 후 retention 뒤 cleanup |
| Faulted row가 자동 삭제되나? | 아니다. Poison은 운영자가 처리 |

Trade-off는 단순하다.

```text
direct publish: latency는 낮지만 silent loss 가능
outbox relay: 작은 async delay를 받아들이고 durable retry 확보
```

사용자에게 보이거나 돈과 관련된 workflow에서는 보통 silent loss가 더 나쁜 실패다.

---

## 11. 마지막 mental model

DB transaction은 두 사실의 commit point다.

1. 비즈니스 상태가 바뀌었다.
2. 시스템은 외부 세계에 integration event를 알려야 한다.

Relay는 DB에서 broker로 가는 위험한 network hop을 담당한다. Durable row에서 시작해 publish되거나 명시적으로 fault될 때까지 재시도한다.

Consumer는 duplicate safety를 담당한다.

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
idempotent consumer             ← 중복 / 순서 뒤바뀜 흡수
```

SQL Server와 RabbitMQ가 하나의 ACID transaction을 공유하게 만들 수는 없다. Transactional outbox pattern은 그걸 필요 없게 만든다. 상태 변경과 publish intent를 atomic하게 만들고, cross-system 작업은 재시도 가능한 백그라운드 프로세스가 끝내게 한다.
