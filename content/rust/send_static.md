+++
title = "Fully Understanding T: Send + Sync + 'static"
date = 2026-07-01

[taxonomies]
categories = ["Post"]
tags = ["post", "Rust"]
+++

When working with threads or async tasks in Rust, this bound appears often.

```rust
T: Send + Sync + 'static
```

At first glance, it can look like it means "`T` must live until the program ends and be sent to another thread." But the actual meaning is different.

In one sentence:

> `T: Send + Sync + 'static` means ownership of `T` can be safely moved to another thread, multiple threads can safely access `T` through shared references, and `T` does not contain short-lived references.

`Send`, `Sync`, and `'static` are independent conditions. Let's separate them one by one.

## `Send`: ownership can be safely moved to another thread

`Send` is a marker trait indicating that a value can be safely moved from one thread to another.

```rust
pub unsafe auto trait Send {}
```

Most common types automatically become `Send` if all of their components are `Send`. For example, `String`, `Vec<T>`, and `Box<T>` can move to another thread when their inner types satisfy the condition.

On the other hand, `Rc<T>` is not `Send`. Its reference count is not updated atomically, so using it across multiple threads is unsafe.

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let value = Rc::new(String::from("hello"));

    thread::spawn(move || {
        println!("{value}");
    });
}
```

This code does not compile. The `move` closure tries to move ownership of `Rc<String>` into a new thread, but `Rc<String>: Send` is not true.

## `Sync`: shared references can be safely accessed by multiple threads

`Sync` is a marker trait indicating that multiple threads can safely use shared references to one value.

```rust
pub unsafe auto trait Sync {}
```

The Rust standard library defines the relationship like this:

> `T` is `Sync` if and only if `&T` is `Send`.

In other words, if it is safe to send `&T` to another thread, then `T: Sync`. `Sync` answers not "can ownership move?" but "can shared references be used by multiple threads?"

`Cell<T>` shows the difference between the two conditions well. `Cell<i32>` is `Send` because its whole ownership can move to another thread, but it is not `Sync` because shared references can mutate the inner value without synchronization.

```rust
use std::cell::Cell;

fn require_send<T: Send>() {}
fn require_sync<T: Sync>() {}

fn main() {
    require_send::<Cell<i32>>();
    require_sync::<Cell<i32>>(); // compile error: Cell<i32> is not Sync.
}
```

When a value needs to be shared across multiple threads, `Arc<T>` is usually used.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let value = Arc::new(String::from("hello"));
    let cloned = Arc::clone(&value);

    let handle = thread::spawn(move || {
        println!("{cloned}");
    });

    handle.join().unwrap();
}
```

However, `Arc<T>` does not automatically make the inner value thread-safe. `Arc` only manages the reference count atomically. For `Arc<T>` to move safely between threads, the inner `T` must also satisfy `Send + Sync`.

If the shared value must be mutated, you also need a tool that synchronizes access itself, such as `Arc<Mutex<T>>`, `Arc<RwLock<T>>`, or atomic types.

The three conditions can appear together in a generic API that sends a shared value to a background thread.

```rust
use std::fmt::Debug;
use std::sync::Arc;
use std::thread::{self, JoinHandle};

fn print_in_background<T>(value: Arc<T>) -> JoinHandle<()>
where
    T: Debug + Send + Sync + 'static,
{
    thread::spawn(move || println!("{value:?}"))
}
```

- `Send`: ownership of `Arc<T>` must be movable into the new thread.
- `Sync`: multiple `Arc`s must be able to access the same `T` through shared references.
- `'static`: `T` must not contain short-lived references into the caller's stack.

## `'static`: it does not mean the value must live forever

`'static` appears differently in two positions.

The first is the lifetime of a reference itself.

```rust
let message: &'static str = "hello";
```

String literals are stored in the program binary, so this reference is valid for the entire execution of the program.

The second is a lifetime bound attached to a type.

```rust
T: 'static
```

This expression does not mean `T` itself must live until the program ends. It means all references inside `T` must not be shorter than `'static`. More practically, it means `T` must not contain short-lived borrowed values.

Owned values usually satisfy this condition naturally.

```rust
fn accept_static<T: 'static>(value: T) {
    drop(value);
}

fn main() {
    let message = String::from("hello");
    accept_static(message);
}
```

`String` owns its data, so `String: 'static`. But the function above drops the value immediately. Therefore, interpreting `T: 'static` as "the value itself lives forever" is wrong.

The following type contains a short-lived reference, so it does not satisfy the `'static` condition.

```rust
fn accept_static<T: 'static>(_: T) {}

fn main() {
    let message = String::from("hello");
    let borrowed = &message;

    accept_static(borrowed);
}
```

The type of `borrowed` is `&String` tied to the lifetime of the local variable `message`. This reference is not `'static`.

## Why does `thread::spawn` show only `Send + 'static`?

Simplified, the signature of `std::thread::spawn` looks like this.

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

A new thread may run longer than the function that called `spawn`. If the `JoinHandle` is dropped, the thread may detach and continue running. The compiler cannot know when the new thread will end, so it cannot allow the closure to leave with short references to local variables. That is why `'static` is required.

The closure itself moves into the new thread. Therefore, values captured by the closure must also be safely movable. That is why `Send` is required.

The return value `T` has the same bounds. `T` is created in the new thread, stored in `JoinHandle<T>`, and then delivered to the thread that calls `join`, so it must be `Send`. A normal thread and `JoinHandle<T>` can move outside the scope where they were created, so `T: 'static` ensures the return value does not contain short references tied to that scope.

This bound is not what prevents returning a reference to a local variable inside the new thread. Such a reference becomes invalid when the function ends, so the borrow checker rejects it regardless of the `'static` bound. In contrast, `ScopedJoinHandle<'scope, T>` cannot leave its scope, so it only requires `T: 'scope` and can return values borrowed from the outer scope.

There is no `Sync` here. If a value is only **moved** into one new thread, multiple threads are not simultaneously accessing it through shared references. For example, `Cell<i32>` is not `Sync`, but the following code is valid because the whole ownership moves into the new thread.

```rust
use std::cell::Cell;
use std::thread;

fn main() {
    let value = Cell::new(42);

    thread::spawn(move || {
        value.set(43);
        assert_eq!(value.get(), 43);
    })
    .join()
    .unwrap();
}
```

So do not add `Sync` to every generic type just because threads are involved. It is needed only when multiple threads actually share the same value.

## `move` does not make a borrowed value `'static`

This is a common failure case.

```rust
use std::thread;

fn main() {
    let message = String::from("hello");
    let borrowed = &message;

    thread::spawn(move || {
        println!("{borrowed}");
    });
}
```

`move` makes the closure capture **variables** by value. But the value captured here is not `String`; it is `&String`. Moving a reference does not extend the lifetime of the data it points to.

Move the owned `String` itself into the closure to fix it.

```rust
use std::thread;

fn main() {
    let message = String::from("hello");

    let handle = thread::spawn(move || {
        println!("{message}");
    });

    handle.join().unwrap();
}
```

Now the closure owns the data directly. `String` satisfies `Send + 'static`, so there is no risk that the reference becomes invalid no matter how long the new thread runs.

## Use `thread::scope` when borrowed values are needed

You do not always need to clone data or wrap it in `Arc`. If you can guarantee that the thread ends within a specific scope, you can use `std::thread::scope`.

```rust
use std::thread;

fn main() {
    let message = String::from("hello");

    thread::scope(|scope| {
        scope.spawn(|| {
            println!("{message}");
        });
    });
}
```

`thread::scope` joins all threads created inside it before leaving the scope. Therefore, the new thread cannot outlive `message`, and the closure can borrow a non-`'static` reference.

The thread boundary still exists, so the `Send` requirement needed for safe cross-thread movement does not disappear. What disappears is the `'static` requirement needed for independently running threads.

## The three conditions are independent

This table helps separate them.

| Type | `Send` | `Sync` | `'static` | Reason |
|---|---:|---:|---:|---|
| `String` | Yes | Yes | Yes | It owns data and can be safely moved or shared. |
| `Rc<String>` | No | No | Yes | It owns data, but the reference count is not thread-safe. |
| `Cell<i32>` | Yes | No | Yes | Ownership can move, but mutation through shared references is not synchronized. |
| `&String` pointing to a local `String` | Yes | Yes | No | The referenced value is safe to share, but the reference is tied to a local value's lifetime. |
| `&'static str` | Yes | Yes | Yes | The referenced data is valid for the whole program execution. |

An owned type is not necessarily `Send` or `Sync`, and a thread-safe reference is not necessarily `'static`.

## The same principle applies in async code

`tokio::spawn` on a multi-threaded runtime also generally requires `Send + 'static` on the future.

- `Send`: the runtime may move the future to another worker thread.
- `'static`: the spawned task may run longer than the current function.

`spawn_local` runs the task only on the current local executor, so it does not require `Send`. But that does not mean the task can freely borrow the caller's stack, so a `'static` bound generally remains.

`tokio::spawn` also does not require the future itself to be `Sync`. However, if the future holds `&T` across an `.await` point and the runtime may move that future to another thread, `T: Sync` may become indirectly required so that the reference is `Send`.

## When should these bounds be added directly?

Habitually adding `Send + Sync + 'static` to generic types unnecessarily restricts the API. Add them only when the actual behavior requires them.

Common cases that require `Send`:

- Moving a value to another OS thread
- A multi-threaded async executor moving work between worker threads
- Passing a value through a thread-safe queue or channel

Common cases that require `Sync`:

- Multiple threads use `&T` at the same time
- The same value is shared across multiple threads or tasks with `Arc<T>`
- One instance is shared by concurrent callers, such as server state, handlers, or callbacks

Common cases that require `'static`:

- Work may outlive the current function call
- A value is stored in a long-lived object or global registry
- A callback or task is stored for later execution

You can understand `Send + 'static` as: "send work to another thread that can run independently of the caller's lifetime." If `Sync` is added too, the contract also says the value can be accessed through shared references by multiple threads.

```rust
use std::thread::{self, JoinHandle};

fn run_in_background<T>(value: T) -> JoinHandle<()>
where
    T: Send + 'static,
{
    thread::spawn(move || drop(value))
}
```

On the other hand, if a function uses a value synchronously and returns, these bounds are usually unnecessary. The function can safely work with short references.

## Decision order when a compiler error appears

When an error related to `Send + Sync + 'static` appears, do not blindly add bounds. Check in this order.

1. **Does the work really need to outlive the caller?** If not, run it directly or consider lifetime-limited APIs such as `thread::scope`.
2. **Can the closure take owned values instead of references?** Move values with `move`, or clone only the necessary pieces.
3. **Do multiple tasks need to share data?** If so, consider concurrency tools such as `Arc`, `Mutex`, or channels.
4. **Why is the type not `Send`?** Check whether it contains values that must not cross thread boundaries, such as `Rc`, thread-local values, or lock guards.
5. **Does the type really need to be `Sync`?** If ownership only moves to one thread, `Sync` may not be needed. If sharing is needed, check for unsynchronized interior mutability.
6. **Is multi-threaded execution really required?** If a single-thread local executor can be used, you may avoid `Send` and `Sync` requirements.

`T: Send + Sync + 'static` is not an incantation to appease the compiler. It is a contract that tells the caller **where ownership of the value can move**, **how the value can be shared**, and **how long it can be stored**.

## References

- [Rust standard library: `Send`](https://doc.rust-lang.org/std/marker/trait.Send.html)
- [Rust standard library: `Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html)
- [Rust standard library: `std::thread::spawn`](https://doc.rust-lang.org/std/thread/fn.spawn.html)
- [Rust standard library: `std::thread::scope`](https://doc.rust-lang.org/std/thread/fn.scope.html)
- [The Rust Reference: Lifetime bounds](https://doc.rust-lang.org/reference/trait-bounds.html#lifetime-bounds)
- [Tokio: `tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html)
