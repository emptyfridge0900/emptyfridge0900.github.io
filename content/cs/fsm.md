+++
title="Finite State Machine"
date=2025-07-15

[taxonomies]
categories = ["CS"]
tags = ["cs"]
+++

An FSM(Finite State Machine) is a model that describes a system using its **current state** and **state transitions based on input**.

The core idea is simple:

> A system is in exactly one state at a time, and when an input or event arrives, it moves to the next state according to predefined rules.

It is called a "finite" state machine because the number of states is finite. For example, a door can only be `Open` or `Closed`, and a turnstile can only be `Locked` or `Unlocked`.

## Components

An FSM is usually described with five components.

| Component | Meaning |
|---|---|
| `S` | Set of possible states |
| `Σ` | Input alphabet or set of events |
| `δ` | Transition function. It takes the current state and input, then decides the next state |
| `s0` | Initial state |
| `F` | Final state or accepting state. Some problems may not need this |

Mathematically, it is written like this:

```text
FSM = (S, Σ, δ, s0, F)
```

The important part is `δ`.

```text
δ(current_state, input) -> next_state
```

In other words, an FSM decides the next state only from "where am I now?" and "what input did I receive?"

## Example: turnstile

Think of a turnstile like the one at a subway gate.

- `Locked`: locked
- `Unlocked`: passable

Assume there are only two inputs.

- `Coin`: insert money
- `Push`: push the gate

The transition table looks like this.

| Current state | Input | Next state | Description |
|---|---|---|---|
| `Locked` | `Coin` | `Unlocked` | Inserting money unlocks it |
| `Locked` | `Push` | `Locked` | Pushing while locked changes nothing |
| `Unlocked` | `Push` | `Locked` | Passing through locks it again |
| `Unlocked` | `Coin` | `Unlocked` | Already unlocked, so it stays unlocked |

In code, it can look like this.

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

In this code, the `switch` acts as the transition function `δ`.

## Why is it useful?

FSMs are useful when complex conditional logic can be organized around "state."

For example, if an order system is represented with multiple flags like `bool isPaid`, `bool isShipped`, and `bool isCanceled`, strange combinations can appear.

```text
isPaid = false
isShipped = true
isCanceled = true
```

This says the order was shipped without payment and was canceled at the same time. That state should not exist in the real domain.

Representing the system as an FSM reduces these impossible combinations.

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

Then transition rules restrict "which event is allowed in which state."

```text
Created --Pay--> Paid
Paid --Ship--> Shipped
Shipped --Deliver--> Delivered
Created --Cancel--> Canceled
Paid --Cancel--> Canceled
```

This explicitly prevents nonsensical flows such as receiving a `Pay` event while already in the `Delivered` state.

## DFA and NFA

When studying FSMs, DFA and NFA usually appear too.

| Type | Meaning |
|---|---|
| DFA(Deterministic Finite Automaton) | Given the current state and input, the next state is always exactly one state |
| NFA(Nondeterministic Finite Automaton) | The same state and input may allow multiple next states, or there may be ε-transitions without input |

Regular expressions are internally connected to these finite automata. A regex can be built into an NFA and, if needed, converted into a DFA for string matching.

However, an FSM has finite memory. It cannot solve problems that require remembering arbitrary depth, such as infinitely nested parentheses. For those cases, stronger models such as pushdown automata, which have a stack, are needed.

## Is C# async/await an FSM?

Short answer: **yes. But it is closer to a compiler-generated coroutine state machine than a domain FSM.**

In C#, an `async` method looks like it runs as written in source code.

```cs
public static async Task<int> CountBytesAsync(HttpClient client, string url)
{
    byte[] bytes = await client.GetByteArrayAsync(url);
    await Task.Delay(100);
    return bytes.Length;
}
```

But the compiler roughly transforms this method into the following structure.

- The original method becomes a thin wrapper that creates a state machine and returns a `Task`.
- The actual method body moves into `MoveNext()`.
- At each `await` point, the state machine stores "where to resume next."
- Local variables that must survive after an `await` become fields of the state machine.
- When the awaited operation completes, its continuation calls `MoveNext()` again.

As pseudocode, it feels like this. The actual compiler output can vary by version and optimization.

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

This is not an exact decompiled result; it is an example that shows the structure. The core idea is still the same. The real compiler uses `AsyncTaskMethodBuilder<T>.AwaitOnCompleted` or `AwaitUnsafeOnCompleted` so that `MoveNext()` is called again when the awaiter completes.

## What actually happens at `await`

When execution reaches `await`, there are two cases.

### 1. If the operation is already complete, keep running

```cs
byte[] bytes = await client.GetByteArrayAsync(url);
```

If the awaiter from `GetByteArrayAsync(url)` is already complete, the state machine does not stop and immediately continues to the next line.

### 2. If the operation is not complete, return for now

If the operation is not finished yet, the following happens:

1. The current position is saved as something like `State = 0`.
2. Local variables and the awaiter needed later are stored in fields.
3. A continuation is registered so `MoveNext()` is called again when the operation completes.
4. Control returns to the caller without blocking the current thread.

So `await` does not block a thread. It pauses method execution and resumes from the suspension point when the awaited operation completes.

## Regular FSM vs async state machine

| Regular FSM | C# async state machine |
|---|---|
| State: domain states like `Locked`, `Unlocked` | State: compiler-generated execution positions like `-1`, `0`, `1`, `-2` |
| Input: events like `Coin`, `Push` | Input: awaited task completion, exception, cancellation |
| Transition function: handwritten `switch` or table | Compiler-generated `MoveNext()` |
| Final state: accepting/final state | `Task` completes as succeeded, faulted, or canceled |
| Purpose: modeling domain rules | implementing suspend/resume for async methods |

Both are FSMs in the sense that "state and event determine the next execution position." But the async/await state machine is not a domain model designed by a person. It is the compiler's transformation of sequential-looking code into a form that can run asynchronously.

## Is Rust async/await also an FSM?

Rust is similar. **Rust `async` blocks and `async fn` are also transformed by the compiler into state machines that implement `Future`.**

However, Rust's execution model is different from C#'s. When you call an `async fn` in Rust, the function body does not start immediately. The call returns an anonymous `Future` value, and the body progresses only when an executor polls that future.

```rust
async fn page_title(url: &str) -> Option<String> {
    let response = get(url).await;
    let text = response.text().await;
    parse_title(&text)
}
```

Conceptually, the code above creates a future like this:

```rust
enum PageTitleFuture {
    Start,
    WaitingResponse,
    WaitingText,
    Done,
}
```

The real rustc output does not expose this exact enum. The Rust Reference says the actual data format of the future returned by an async block is unspecified. Conceptually, though, it is similar to **an enum where each `await` point has a variant, and that variant stores the data needed to resume**.

The core interface of a Rust future is `Future::poll`.

```rust
trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Self::Output>;
}
```

`poll()` returns one of two values.

| Return value | Meaning |
|---|---|
| `Poll::Pending` | Not finished yet. Poll again later |
| `Poll::Ready(value)` | Finished. `value` is the final result |

If a future reaches an `.await` point and the child future is not ready yet, the current future stores its state and necessary locals, then returns `Poll::Pending`. Later, when I/O or a timer becomes ready, a `Waker` wakes the task and the executor calls `poll()` again. At that point, the future resumes from the stored state instead of starting from the beginning.

So Rust `.await` also does not block a thread. The current future returns `Poll::Pending` and gives control back to the executor, which can poll other tasks on the same thread.

## C# vs Rust async state machines

| Topic | C# | Rust |
|---|---|---|
| When async execution starts | The method may start and run until the first incomplete `await` | Calling `async fn` does not execute the body; it returns a `Future` |
| Return type | `Task`, `Task<T>`, `ValueTask<T>`, etc. | Anonymous `impl Future<Output = T>` type |
| Execution entry point | Compiler-generated `MoveNext()` | `Future::poll()` |
| Resume mechanism | Awaiter completion invokes a continuation | `Waker` wakes the task and the executor polls again |
| Stored data | Locals needed after await, awaiters, state value | Locals needed after await, child futures, state value |
| Notable detail | Task model fits runtime/GC environments well | `Pin` matters because a future may internally reference its own fields |

Both are compiler-generated state machines. The difference is that C# is usually explained through `Task` and continuation, while Rust is explained through `Future`, `Poll`, `Waker`, and executor.

## Things to remember

- An FSM is a tool for understanding complex flows by splitting them into states and transitions.
- The number of states must be finite.
- A transition is "current state + input -> next state."
- A C# `async` method is compiled into a structure that implements `IAsyncStateMachine`.
- Rust `async` blocks and `async fn` are also compiled into state machines that implement `Future`.
- `await` does not block a thread. It suspends the method and resumes it through a continuation when the operation completes.
- The state value in an async state machine is not a domain state. It is an execution position that says "which await point did we stop at?"

## References

- Finite-state machine: <https://en.wikipedia.org/wiki/Finite-state_machine>
- Regex and automata: <https://statkclee.github.io/nlp2/regex-under-the-hood.html>
- C# language specification - Async functions: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#1514-async-functions>
- `await` operator: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await>
- `IAsyncStateMachine`: <https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.iasyncstatemachine>
- `AsyncStateMachineAttribute`: <https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.asyncstatemachineattribute>
- How Async/Await Really Works in C#: <https://devblogs.microsoft.com/dotnet/how-async-await-really-works/>
- Rust Reference - async blocks: <https://doc.rust-lang.org/reference/expressions/block-expr.html#async-blocks>
- Rust Reference - async functions: <https://doc.rust-lang.org/reference/items/functions.html#async-functions>
- Rust `Future` trait: <https://doc.rust-lang.org/std/future/trait.Future.html>
- Asynchronous Programming in Rust - async/.await primer: <https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html>
