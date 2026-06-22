+++
title = "선빵친놈이 이긴다 — RabbitMQ Exchange 상태 불일치 디버깅"
date = 2026-06-22

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ", "MassTransit", "debugging", "messaging"]
+++

프로덕션에서 WeChat inbound 메시지 경로가 통째로 끊겼다. 코드를 뒤졌는데 손댈 곳이 단 한 줄도 없었다. 범인은 **브로커에 남아 있던 잘못된 exchange 정의**였고, 그걸 가능하게 한 건 RabbitMQ의 한 가지 핵심 속성이었다.

## 증상

프로덕션 아래와 같은 에러가 무한 반복되고 있었다.

```
MassTransit.RabbitMqConnectionException: ReceiveTransport faulted ...
RabbitMQ.Client.Exceptions.OperationInterruptedException:
  code=406, text='PRECONDITION_FAILED - inequivalent arg 'durable'
  for exchange '...:DispatchInboundWechatMessageCreated'
  in vhost '/': received 'true' but current is 'false''
```

해석하면 이렇다.

- 우리 앱이 exchange를 **`durable=true`로 선언** 하려고 한다 (`received 'true'`).
- 브로커에 **이미 존재하는** 같은 이름의 exchange는 **`durable=false`** 다 (`current is 'false'`).
- RabbitMQ: "둘이 다르잖아. 못 받아줘." → `406 PRECONDITION_FAILED` → 채널 종료.

결과적으로 MassTransit의 receive transport가 fault → 재연결 → 또 fault 루프에 빠지면서 **WeChat inbound 메시지 처리가 전혀 안 됐다.**

## 직관적으로 — 이게 대체 무슨 상황인가

벽에 콘센트(exchange)가 하나 박혀 있다고 하자. 이 콘센트는 **처음 설치한 사람이 규격을 정해서 못으로 박아버린다.** 한 번 박히면 규격을 바꿀 수 없다.

- 우리 앱은 **220V 규격(durable)** 콘센트를 기대하고 플러그를 꽂으려 한다.
- 누군가가 예전에 이 자리에 **110V 규격(non-durable)** 콘센트를 먼저 박아놨다.
- 플러그가 안 맞는다. 그렇다고 RabbitMQ가 콘센트를 220V로 바꿔주지도 않는다. 그냥 "규격 안 맞음"이라고 채널을 끊어버린다.

핵심 직관 두 가지:

1. **exchange 규격은 처음 선언한 사람이 정하고, 그 뒤로는 불변이다.** → first-declarer-wins.
2. **규격이 안 맞으면 RabbitMQ는 고쳐주는 게 아니라 거절한다.** → `PRECONDITION_FAILED`.

"코드를 고친다"고 풀리는 문제가 아니다. **브로커에 잘못 남은 exchange를 뽑아내야** 우리 앱이 올바른 규격으로 다시 만들 수 있다.

## 범인이 된 RabbitMQ 속성들

### 선언 멱등성 + 동등성 검사 (가장 핵심)

`exchange.declare`는 **멱등(idempotent)** 이다. 같은 걸 여러 번 선언해도 문제없다 — 인자가 똑같을 때만. 이미 존재하는 exchange를 **다른 인자**로 선언하면 수정이 아니라 거절이다.

> "이미 있는 거랑 인자가 다르네. 안 해줄래." → `406 PRECONDITION_FAILED`, 채널 종료.

exchange의 속성(`type`, `durable`, `auto_delete`, `internal`, `arguments`)은 **생성 시점에 고정, 이후 변경 불가.** 바꾸려면 지우고 다시 만드는 수밖에 없다.

### Durable vs Transient

| | 설명 |
|---|---|
| `durable=true` | exchange 정의가 디스크에 저장 → **브로커 재시작 후에도 살아남는다** |
| `durable=false` | 메모리에만 존재 → 브로커 재시작 시 사라진다 |

문제의 exchange는 `durable=false`였고, 우리 앱은 `durable=true`를 원했다.

### Auto-delete

`auto-delete=true`이면 마지막 바인딩이 사라질 때 exchange가 스스로 삭제된다. 관리 UI에서 Features 컬럼에 `AD`로 표시된다. 문제의 exchange는 `AD` + non-durable 조합 — **"임시 exchange"의 전형적인 모양이다.**

### Topology 영속성

exchange/queue/binding은 **서버(브로커) 측 상태**다. 그걸 만든 클라이언트 연결이 끊겨도 정의는 브로커에 남는다. 브로커 컨테이너 수명이 `Persistent`라서, 한 번 잘못 만들어진 정의가 계속 살아남아 있었다.

## 왜 이게 코드 버그가 아닌가

처음엔 "코드 어딘가에서 `durable=false`로 선언했겠지"라고 의심했다. 두 가지가 그 가설을 깨뜨렸다.

### 증거 1: git 전체 히스토리에 durability 설정이 단 한 번도 없었다

```bash
git log --all -S "Durable" -- '*.cs'                                    # → (없음)
git log --all -G "Durable|AutoDelete|Temporary|PublishTopology" -- '*.cs'  # → (없음)
```

durability/auto-delete를 설정하거나 바꾼 커밋이 역사상 존재하지 않았다. 코드는 항상 MassTransit의 기본값(=durable)만 썼다.

### 증거 2: WeChat만 깨지고 WhatsApp/LINE는 멀쩡했다

WeChat, WhatsApp, LINE, Internal 네 컨텍스트의 inbound dispatch 코드는 **구조적으로 완전히 동일**하다.

> 코드/설정 문제였다면 네 개가 **똑같이** 깨졌어야 한다. 그런데 WeChat 하나만 깨졌다.

이 비대칭이 코드 원인설을 확정적으로 배제한다. **이건 코드 버그도 설계 결함도 아니다. 브로커에 남은 잘못된 exchange 정의와 코드 선언 사이의 "상태 불일치(state mismatch)"다.**

## 왜 하필 WeChat만? — first-declarer-wins는 exchange 단위

durability 고정이 exchange **하나하나** 단위로 일어난다는 게 핵심이다.

- WhatsApp/LINE/Internal exchange: 처음 선언한 게 우리 앱(durable) → durable로 굳음 → 평화.
- WeChat exchange: **처음 선언한 누군가가 transient(non-durable+auto-delete)** 였다 → AD로 굳음 → 이후 우리 앱의 durable 선언이 매번 충돌.

벽에 콘센트 네 개를 박는데 세 개는 규격대로 박혔고, 하나만 다른 사람이 먼저 엉뚱한 규격으로 박아버린 것이다. 설계도(코드)는 네 개 다 똑같다.

> **WhatsApp과 LINE은 "안전"한 게 아니라 "운이 좋았던" 것이다.** 잘못된 클라이언트가 그들의 exchange를 먼저 transient로 선언하면 똑같이 깨진다.

## 진단 — 범인이 아직 살아있나?

지우기 전에 한 가지 확인이 필요했다. 시나리오가 두 가지였기 때문이다.

- **(A) 살아 있는 외부 publisher**: 지금도 이 exchange를 transient로 계속 다시 만든다 → 내가 지워도 곧바로 다시 AD로 생긴다. 먼저 그 클라이언트를 멈춰야 한다.
- **(B) 오래된 잔재(orphaned leftover)**: 과거에 누군가 한 번 만들어 놨고 그 클라이언트는 이미 사라졌다 → 한 번 지우면 끝.

브로커의 연결 목록으로 확인했다.

```bash
rabbitmqctl list_connections name peer_host user channels
```

연결은 **딱 둘**, 둘 다 우리 앱(채널 59개 = MassTransit 풀 버스, 두 개의 replica)이었다. 제3의 의심스러운 연결 없음 → **(B) 오래된 잔재 케이스.** 그냥 지우면 된다.

## 해결 — 지우면 앱이 알아서 다시 만든다

```bash
rabbitmqadmin -u "$RABBITMQ_DEFAULT_USER" -p "$RABBITMQ_DEFAULT_PASS" --vhost / \
  delete exchange --name '...:DispatchInboundWechatMessageCreated'
```

지운 직후, MassTransit의 재연결 루프가 다음 시도에서 exchange를 **durable로 다시 선언**했다. 충돌할 대상이 없으니 성공이다.

```bash
rabbitmqctl list_exchanges name durable auto_delete | grep DispatchInboundWechat
# → ... true  false   (durable=true, auto_delete=false) ✅
```

관리 UI에서도 Features가 `D`로 돌아왔다. `precondition_failed` 로그가 멈췄고, `ReceiveTransport faulted ... Retrying` 루프도 멈췄다.

exchange 삭제는 **queue와 그 안의 메시지를 건드리지 않는다.** exchange와 binding만 사라지고, MassTransit이 binding도 자동으로 재생성한다.

## 재발 방지

코드 리뷰로는 절대 못 잡는 종류의 문제(브로커 상태)이기 때문에, **런타임 감시 장치**가 필요했다.

`RabbitMqExchangeTopologyGuard`를 추가했다.
- 시작 시 + 10분마다 도는 read-only `BackgroundService`
- RabbitMQ management API(`/api/exchanges/{vhost}`)를 GET해서 `Omni.` 접두 exchange 중 **non-durable이거나 auto-delete인 것**을 `LogWarning`으로 띄운다
- `bus-*`, `amq.*` 같은 정상적인 임시 endpoint는 제외 (MassTransit의 임시 bus/reply endpoint는 원래 AD가 맞다)
- **브로커를 건드리지 않는다.** 실제 복구는 삭제→재선언으로 수동 처리

이 가드가 **"감지"는 하지만 "예방/자동복구"는 못 한다.** 진짜로 재발을 없애려면 추가로:
1. `LogWarning`을 **실제 알림/health check로 승격** — 로그만으로는 묻힌다.
2. 드리프트 발생 시 `/api/connections`까지 조회해 범인 연결을 같이 로깅.
3. **환경 점검**: prod 브로커 연결 문자열을 들고 있는 non-prod 인스턴스가 없는지 감사.

## 교훈

1. **코드에 없으면 코드 문제가 아니다.** git pickaxe(`-S`/`-G`)로 "그 설정을 만진 커밋이 존재하는가"를 먼저 확인하면, 엉뚱한 곳을 파는 시간을 아낀다.

2. **분산 시스템 버그는 "코드"가 아니라 "상태"에 있을 수 있다.** 브로커, DB, 캐시처럼 오래 사는 외부 상태는 코드와 별개의 진실을 갖는다.

3. **비대칭은 강력한 진단 신호다.** "똑같은 코드인데 왜 하나만?" → 원인이 코드/설정(대칭)이 아니라 인스턴스별 상태(비대칭)에 있다는 뜻이다.

4. **first-declarer-wins를 기억하라.** RabbitMQ exchange/queue 인자는 최초 선언 시 고정 + 불변이다. 같은 브로커를 여러 클라이언트가 공유하면, 누가 먼저 선언하느냐로 영구적 차이가 생긴다.

5. **"안 깨진 것"과 "안전한 것"은 다르다.** WhatsApp/LINE는 구조적으로 보호된 게 아니라 운이 좋았을 뿐이다.

---

## 3줄 요약

1. RabbitMQ exchange의 속성(durable, auto-delete 등)은 **최초 선언 시 고정, 이후 불변**이다 — 다른 인자로 재선언하면 수정이 아니라 `406 PRECONDITION_FAILED`로 채널이 끊긴다(first-declarer-wins).
2. WeChat exchange만 깨진 건 코드 문제가 아니라 **상태 불일치**였다 — 과거에 누군가 transient로 선언한 잔재가 브로커에 남아 우리 앱의 durable 선언을 매번 막고 있었다.
3. 해결은 잘못된 exchange 삭제 하나로 끝났다; `list_connections`로 "살아 있는 재선언자가 없음"을 먼저 확인한 뒤 지우면 앱이 알아서 durable로 재선언한다.
