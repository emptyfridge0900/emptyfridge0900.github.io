+++
title="Finite State Machine"
date=2025-07-15

[taxonomies]
categories = ["CS"]
tags = ["cs"]
+++

FSM(Finite State Machine, 유한 상태 기계)은 어떤 시스템을 **현재 상태**와 **입력에
따른 상태 전이**로 설명하는 모델이다.

핵심은 단순하다:

> 시스템은 한 번에 하나의 상태에 있고, 입력이나 이벤트가 들어오면 정해진 규칙에
> 따라 다음 상태로 이동한다.

상태의 개수가 유한하기 때문에 "finite" state machine이라고 부른다. 예를 들어 문은
`Open` 또는 `Closed` 상태만 가질 수 있고, 회전문(turnstile)은 `Locked` 또는
`Unlocked` 상태만 가질 수 있다.

## 구성 요소

FSM은 보통 다음 5가지로 설명한다.

| 구성 요소 | 의미 |
|---|---|
| `S` | 가능한 상태들의 집합 |
| `Σ` | 입력 알파벳 또는 이벤트들의 집합 |
| `δ` | 전이 함수. 현재 상태와 입력을 받아 다음 상태를 결정한다 |
| `s0` | 시작 상태 |
| `F` | 종료 상태 또는 accepting state. 문제에 따라 없을 수도 있다 |

수식으로는 이렇게 쓴다:

```text
FSM = (S, Σ, δ, s0, F)
```

중요한 부분은 `δ`다.

```text
δ(current_state, input) -> next_state
```

즉 FSM은 "지금 어디에 있는가?"와 "무슨 입력을 받았는가?"만으로 다음 상태를 정한다.

## 예제: 회전문

지하철 개찰구 같은 회전문을 생각해보자.

- `Locked`: 잠겨 있음
- `Unlocked`: 통과 가능

입력은 두 개만 있다고 하자.

- `Coin`: 돈을 넣음
- `Push`: 문을 밀음

상태 전이표는 이렇게 된다.

| 현재 상태 | 입력 | 다음 상태 | 설명 |
|---|---|---|---|
| `Locked` | `Coin` | `Unlocked` | 돈을 넣으면 통과 가능 |
| `Locked` | `Push` | `Locked` | 잠긴 상태에서 밀어도 그대로 |
| `Unlocked` | `Push` | `Locked` | 통과하면 다시 잠김 |
| `Unlocked` | `Coin` | `Unlocked` | 이미 열려 있으니 그대로 |

코드로 쓰면 이런 모양이다.

```cs
enum TurnstileState
{
    Locked,
    Unlocked
}

enum TurnstileEvent
{
    Coin,
    Push
}

static TurnstileState Next(TurnstileState state, TurnstileEvent input) =>
    (state, input) switch
    {
        (TurnstileState.Locked, TurnstileEvent.Coin) => TurnstileState.Unlocked,
        (TurnstileState.Locked, TurnstileEvent.Push) => TurnstileState.Locked,
        (TurnstileState.Unlocked, TurnstileEvent.Push) => TurnstileState.Locked,
        (TurnstileState.Unlocked, TurnstileEvent.Coin) => TurnstileState.Unlocked,
        _ => throw new InvalidOperationException("Unknown transition")
    };
```

이 코드에서 `switch`가 전이 함수 `δ` 역할을 한다.

## 왜 유용한가

FSM은 복잡한 조건문을 "상태" 중심으로 정리할 때 좋다.

예를 들어 주문 시스템을 `bool isPaid`, `bool isShipped`, `bool isCanceled` 같은 플래그
여러 개로 표현하면 이상한 조합이 생길 수 있다.

```text
isPaid = false
isShipped = true
isCanceled = true
```

결제도 안 됐는데 배송됐고, 동시에 취소도 된 상태다. 이런 상태는 실제 도메인에서는
존재하면 안 된다.

FSM으로 표현하면 이런 불가능한 조합을 줄일 수 있다.

```cs
enum OrderState
{
    Created,
    Paid,
    Shipped,
    Delivered,
    Canceled
}
```

그리고 "어떤 상태에서 어떤 이벤트가 가능한가"를 전이 규칙으로 제한한다.

```text
Created --Pay--> Paid
Paid --Ship--> Shipped
Shipped --Deliver--> Delivered
Created --Cancel--> Canceled
Paid --Cancel--> Canceled
```

이렇게 하면 `Delivered` 상태에서 `Pay` 이벤트가 들어오는 것처럼 말이 안 되는 흐름을
명시적으로 막을 수 있다.

## DFA와 NFA

FSM을 공부하면 보통 DFA와 NFA를 같이 보게 된다.

| 종류 | 의미 |
|---|---|
| DFA(Deterministic Finite Automaton) | 현재 상태와 입력이 정해지면 다음 상태가 항상 하나로 결정된다 |
| NFA(Nondeterministic Finite Automaton) | 같은 상태와 입력에서 여러 다음 상태가 가능하거나, 입력 없이 이동하는 ε-transition이 있을 수 있다 |

정규 표현식(regex)은 내부적으로 이런 유한 오토마타와 연결된다. 보통 정규식을 NFA로
만들고, 필요하면 DFA로 변환해서 문자열을 매칭할 수 있다.

다만 FSM은 메모리가 유한하다. 그래서 임의 깊이로 중첩되는 괄호처럼 "얼마나 깊게
들어갔는지"를 무한히 기억해야 하는 문제는 순수 FSM만으로 처리할 수 없다. 이런 경우에는
스택을 가진 pushdown automaton 같은 더 강한 모델이 필요하다.

## C# async/await도 FSM인가?

결론부터 말하면, **그렇다. 하지만 도메인 FSM이라기보다는 컴파일러가 만든 coroutine
state machine에 가깝다.**

C#에서 `async` 메서드는 소스 코드 그대로 실행되는 것처럼 보인다.

```cs
public static async Task<int> CountBytesAsync(HttpClient client, string url)
{
    byte[] bytes = await client.GetByteArrayAsync(url);
    await Task.Delay(100);
    return bytes.Length;
}
```

하지만 컴파일러는 이 메서드를 대략 다음 구조로 바꾼다.

- 원래 메서드는 state machine을 만들고 `Task`를 반환하는 얇은 wrapper가 된다.
- 실제 메서드 본문은 `MoveNext()` 안으로 들어간다.
- `await` 지점마다 "다음에 어디서 다시 시작해야 하는지"를 상태 값으로 저장한다.
- `await` 이후에도 살아 있어야 하는 지역 변수는 state machine의 필드로 올라간다.
- 기다리던 작업이 끝나면 continuation이 `MoveNext()`를 다시 호출한다.

의사 코드로 보면 이런 느낌이다. 실제 컴파일러 출력은 버전과 최적화에 따라 달라질 수
있다.

```cs
private struct CountBytesAsyncStateMachine : IAsyncStateMachine
{
    public int State;
    public AsyncTaskMethodBuilder<int> Builder;
    public HttpClient Client;
    public string Url;

    private byte[] _bytes;
    private TaskAwaiter<byte[]> _downloadAwaiter;
    private TaskAwaiter _delayAwaiter;

    public void MoveNext()
    {
        try
        {
            TaskAwaiter<byte[]> downloadAwaiter;

            if (State == 0)
            {
                downloadAwaiter = _downloadAwaiter;
                _downloadAwaiter = default;
                State = -1;
                goto AfterDownload;
            }

            downloadAwaiter = Client.GetByteArrayAsync(Url).GetAwaiter();

            if (!downloadAwaiter.IsCompleted)
            {
                State = 0;
                _downloadAwaiter = downloadAwaiter;
                RegisterContinuation(downloadAwaiter, MoveNext);
                return;
            }

        AfterDownload:
            _bytes = downloadAwaiter.GetResult();

            TaskAwaiter delayAwaiter;

            if (State == 1)
            {
                delayAwaiter = _delayAwaiter;
                _delayAwaiter = default;
                State = -1;
                goto AfterDelay;
            }

            delayAwaiter = Task.Delay(100).GetAwaiter();

            if (!delayAwaiter.IsCompleted)
            {
                State = 1;
                _delayAwaiter = delayAwaiter;
                RegisterContinuation(delayAwaiter, MoveNext);
                return;
            }

        AfterDelay:
            delayAwaiter.GetResult();

            State = -2;
            Builder.SetResult(_bytes.Length);
        }
        catch (Exception ex)
        {
            State = -2;
            Builder.SetException(ex);
        }
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

위 코드는 정확한 디컴파일 결과가 아니라 구조를 보여주기 위한 예시다. 그래도 핵심은
실제와 같다. 실제 컴파일러는 `AsyncTaskMethodBuilder<T>`의 `AwaitOnCompleted` 또는
`AwaitUnsafeOnCompleted`를 사용해서 awaiter 완료 시 `MoveNext()`가 다시 호출되도록
연결한다.

## await에서 실제로 일어나는 일

`await`를 만나면 두 가지 경우가 있다.

### 1. 이미 완료된 작업이면 계속 실행

```cs
byte[] bytes = await client.GetByteArrayAsync(url);
```

`GetByteArrayAsync(url)`의 awaiter가 이미 완료되어 있으면 state machine은 멈추지 않고
바로 다음 줄로 진행한다.

### 2. 아직 완료되지 않았으면 일단 반환

작업이 아직 끝나지 않았다면 다음 일이 일어난다.

1. 현재 위치를 `State = 0` 같은 값으로 저장한다.
2. 나중에 다시 필요할 지역 변수와 awaiter를 필드에 저장한다.
3. 작업이 끝났을 때 `MoveNext()`를 다시 호출할 continuation을 등록한다.
4. 현재 스레드를 막지 않고 caller에게 제어권을 돌려준다.

그래서 `await`는 thread를 block하는 것이 아니다. `await`는 메서드의 실행을 잠시
중단하고, 기다리던 작업이 끝나면 중단 지점 다음부터 다시 실행되게 만든다.

## 일반 FSM과 async state machine 비교

| 일반 FSM | C# async state machine |
|---|---|
| 상태: `Locked`, `Unlocked` 같은 도메인 상태 | 상태: `-1`, `0`, `1`, `-2` 같은 compiler-generated 실행 위치 |
| 입력: `Coin`, `Push` 같은 이벤트 | 입력: awaited task completion, exception, cancellation |
| 전이 함수: 직접 작성한 `switch` 또는 table | 컴파일러가 생성한 `MoveNext()` |
| 종료 상태: accepting/final state | `Task`가 succeeded, faulted, canceled 중 하나로 완료 |
| 목적: 도메인 규칙 모델링 | async 메서드의 suspend/resume 구현 |

즉 둘 다 "상태와 이벤트에 따라 다음 실행 위치가 결정된다"는 점에서는 FSM이다. 하지만
async/await의 state machine은 사람이 직접 설계하는 도메인 모델이 아니라, 사람이 쓴
순차적인 코드를 컴파일러가 비동기 실행 가능한 형태로 바꾼 결과다.

## Rust async/await도 FSM인가?

Rust도 마찬가지다. **Rust의 `async` 블록과 `async fn`도 컴파일러가 `Future`를
구현하는 state machine으로 바꾼다.**

다만 C#과 실행 모델이 조금 다르다. Rust에서 `async fn`을 호출하면 함수 본문이 바로
실행되지 않는다. 호출 결과는 익명 타입의 `Future` 값이고, 이 future가 executor에 의해
`poll()`될 때 실제 본문이 진행된다.

```rust
async fn page_title(url: &str) -> Option<String> {
    let response = get(url).await;
    let text = response.text().await;
    parse_title(&text)
}
```

위 코드는 개념적으로 이런 future를 만든다고 볼 수 있다.

```rust
enum PageTitleFuture {
    Start,
    WaitingResponse,
    WaitingText,
    Done,
}
```

실제 rustc가 저 enum을 그대로 노출하는 것은 아니다. Rust Reference는 async block이
반환하는 future 타입의 실제 데이터 형식은 지정하지 않는다고 말한다. 하지만 개념적으로는
**각 `await` 지점마다 하나의 variant가 있고, 그 variant 안에 resume에 필요한 데이터를
저장하는 enum**과 비슷하다고 설명한다.

Rust future의 핵심 인터페이스는 `Future::poll`이다.

```rust
trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Self::Output>;
}
```

`poll()`은 다음 둘 중 하나를 반환한다.

| 반환값 | 의미 |
|---|---|
| `Poll::Pending` | 아직 끝나지 않았다. 나중에 다시 poll해야 한다 |
| `Poll::Ready(value)` | 완료됐다. 최종 결과가 `value`다 |

어떤 `.await` 지점에서 아직 준비되지 않은 future를 만나면, 현재 future는 자기 상태와
필요한 지역 변수를 저장한 뒤 `Poll::Pending`을 반환한다. 나중에 I/O나 타이머 같은 작업이
준비되면 `Waker`가 task를 깨우고, executor가 다시 `poll()`을 호출한다. 그때 future는
처음부터 다시 시작하는 것이 아니라 저장된 상태에서 이어서 실행한다.

그래서 Rust의 `.await`도 thread를 block하는 것이 아니다. 현재 future가
`Poll::Pending`을 반환해서 executor에게 제어권을 돌려주고, executor는 같은 thread에서
다른 task를 poll할 수 있다.

## C#과 Rust async state machine 비교

| 항목 | C# | Rust |
|---|---|---|
| async 호출 시점 | 메서드가 시작되고 첫 incomplete `await`까지 진행될 수 있다 | `async fn`은 호출만으로 본문을 실행하지 않고 `Future`를 반환한다 |
| 반환 타입 | `Task`, `Task<T>`, `ValueTask<T>` 등 | 익명 타입의 `impl Future<Output = T>` |
| 실행 진입점 | compiler-generated `MoveNext()` | `Future::poll()` |
| 재개 방식 | awaiter completion이 continuation을 호출한다 | `Waker`가 task를 깨우고 executor가 다시 poll한다 |
| 저장되는 것 | await 이후 필요한 locals, awaiter, 상태 값 | await 이후 필요한 locals, child future, 상태 값 |
| 특이점 | 런타임/GC 환경과 잘 맞는 task 모델 | `Pin`이 중요하다. future 내부가 자기 자신의 필드를 참조할 수 있기 때문이다 |

둘 다 compiler-generated state machine이라는 점은 같다. 차이는 C#은 `Task`와
continuation 중심으로 설명되고, Rust는 `Future`, `Poll`, `Waker`, executor 중심으로
설명된다는 점이다.

## 기억할 점

- FSM은 복잡한 흐름을 상태와 전이로 쪼개서 이해하는 도구다.
- 상태는 유한해야 한다.
- 전이는 "현재 상태 + 입력 -> 다음 상태"로 정리한다.
- C#의 `async` 메서드는 컴파일될 때 `IAsyncStateMachine`을 구현하는 구조로 바뀐다.
- Rust의 `async` 블록과 `async fn`도 `Future`를 구현하는 state machine으로 바뀐다.
- `await`는 스레드를 block하지 않는다. 메서드를 suspend하고, 작업 완료 후
  continuation으로 resume한다.
- async state machine의 상태 값은 도메인 상태가 아니라 "어느 await 지점에서
  멈췄는가"를 나타내는 실행 위치다.

## 참고

- 유한 상태 기계: <https://ko.wikipedia.org/wiki/%EC%9C%A0%ED%95%9C_%EC%83%81%ED%83%9C_%EA%B8%B0%EA%B3%84>
- 정규식과 오토마타: <https://statkclee.github.io/nlp2/regex-under-the-hood.html>
- C# language specification - Async functions: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#1514-async-functions>
- `await` operator: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await>
- `IAsyncStateMachine`: <https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.iasyncstatemachine>
- `AsyncStateMachineAttribute`: <https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.asyncstatemachineattribute>
- How Async/Await Really Works in C#: <https://devblogs.microsoft.com/dotnet/how-async-await-really-works/>
- Rust Reference - async blocks: <https://doc.rust-lang.org/reference/expressions/block-expr.html#async-blocks>
- Rust Reference - async functions: <https://doc.rust-lang.org/reference/items/functions.html#async-functions>
- Rust `Future` trait: <https://doc.rust-lang.org/std/future/trait.Future.html>
- Asynchronous Programming in Rust - async/.await primer: <https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html>
