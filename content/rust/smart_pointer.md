+++
title="Smart Pointer"
date=2025-05-24

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

Rust does not have a single `SmartPointer` trait. Instead, the term describes types that own or wrap a value and provide pointer-like behavior, usually through [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html) and [`Drop`](https://doc.rust-lang.org/std/ops/trait.Drop.html).

Some types below, such as `RefCell<T>`, `Mutex<T>`, and `Pin<P>`, are more precisely interior-mutability or pointer-wrapper types. They are included because they are commonly combined with owning smart pointers in real Rust programs.

## [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html)

`Box<T>` owns one value stored on the heap. Use it when:

- a type's size cannot be known at compile time, such as a recursive type or trait object;
- a large value must move without copying its heap allocation; or
- one component should have exclusive ownership of a heap value.

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));
```

The `Box` gives the recursive field a known size: it stores a fixed-size pointer rather than embedding another `List` directly.

## [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html)

`Rc<T>` provides shared ownership through reference counting in a single thread. Cloning an `Rc` increments the reference count; it does not clone the underlying value.

```rust
use std::rc::Rc;

let name = Rc::new(String::from("Ferris"));
let first_owner = Rc::clone(&name);
let second_owner = Rc::clone(&name);

assert_eq!(first_owner.as_str(), "Ferris");
assert_eq!(second_owner.as_str(), "Ferris");
assert_eq!(Rc::strong_count(&name), 3);
```

Use `Rc<T>` for shared read-only data in single-threaded structures such as syntax trees, GUI trees, and graph nodes. It is neither `Send` nor `Sync`; use `Arc<T>` when ownership must cross thread boundaries.

## [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html)

`Arc<T>` is the atomic, thread-safe counterpart of `Rc<T>`. Its atomic reference counting has a small synchronization cost, so prefer `Rc<T>` when threads are not involved.

```rust
use std::sync::Arc;
use std::thread;

let values = Arc::new(vec![10, 20, 30]);
let mut handles = Vec::new();

for index in 0..values.len() {
    let values = Arc::clone(&values);
    handles.push(thread::spawn(move || values[index]));
}

let results: Vec<i32> = handles
    .into_iter()
    .map(|handle| handle.join().unwrap())
    .collect();

assert_eq!(results, vec![10, 20, 30]);
```

`Arc<T>` shares ownership, not mutability. To mutate shared state, it is commonly combined with `Mutex<T>` or `RwLock<T>`.

## [`RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html)

`RefCell<T>` provides interior mutability in a single thread. It enforces Rust's borrowing rules at runtime instead of compile time: one mutable borrow or any number of immutable borrows may exist, but not both. Violating that rule causes a panic.

```rust
use std::cell::RefCell;

let message = RefCell::new(String::from("Hello"));

message.borrow_mut().push_str(", Rust");

assert_eq!(message.borrow().as_str(), "Hello, Rust");
```

The common `Rc<RefCell<T>>` combination gives multiple single-threaded owners mutable access to the same value. Keep each `borrow()` or `borrow_mut()` scope short to avoid runtime borrow conflicts.

## [`Weak<T>`](https://doc.rust-lang.org/std/rc/struct.Weak.html)

`Weak<T>` is a non-owning reference to an allocation managed by `Rc<T>`. The corresponding [`std::sync::Weak<T>`](https://doc.rust-lang.org/std/sync/struct.Weak.html) works with `Arc<T>`.

A weak reference does not keep the value alive. Calling `upgrade()` returns `Some(Rc<T>)` while a strong owner exists and `None` after the value has been dropped.

```rust
use std::rc::Rc;

let owner = Rc::new(String::from("cached value"));
let observer = Rc::downgrade(&owner);

assert_eq!(observer.upgrade().unwrap().as_str(), "cached value");

drop(owner);
assert!(observer.upgrade().is_none());
```

Use `Weak<T>` for back-references, caches, observers, and parent pointers. It is also the standard way to break `Rc` or `Arc` reference cycles that would otherwise leak memory.

## [`Cell<T>`](https://doc.rust-lang.org/std/cell/struct.Cell.html)

`Cell<T>` is lightweight single-threaded interior mutability. Unlike `RefCell<T>`, it does not return references to its inner value. Values are copied out with `get()` when `T: Copy`, or replaced as a whole.

```rust
use std::cell::Cell;

let request_count = Cell::new(0_u32);

request_count.set(request_count.get() + 1);
request_count.set(request_count.get() + 1);

assert_eq!(request_count.get(), 2);
```

Use it for small `Copy` values such as counters, flags, and enum states. Use `RefCell<T>` when code needs borrowed access to a larger or non-`Copy` value.

## [`Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html)

`Mutex<T>` gives one thread mutable access at a time. Locking returns a guard that unlocks automatically when dropped. Shared ownership normally comes from wrapping it in an `Arc`.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0_u32));
let mut handles = Vec::new();

for _ in 0..4 {
    let counter = Arc::clone(&counter);
    handles.push(thread::spawn(move || {
        *counter.lock().unwrap() += 1;
    }));
}

for handle in handles {
    handle.join().unwrap();
}

assert_eq!(*counter.lock().unwrap(), 4);
```

Do not hold a mutex guard across slow work or an `.await` point. In asynchronous code, use the async runtime's synchronization primitives when the guard must survive across `.await`.

## [`RwLock<T>`](https://doc.rust-lang.org/std/sync/struct.RwLock.html)

`RwLock<T>` allows either multiple readers or one writer at a time. It can improve concurrency for read-heavy workloads, but it has more overhead than `Mutex<T>` and its scheduling policy depends on the operating system.

```rust
use std::sync::RwLock;

let configuration = RwLock::new(String::from("development"));

{
    let first_reader = configuration.read().unwrap();
    let second_reader = configuration.read().unwrap();
    assert_eq!(*first_reader, *second_reader);
}

*configuration.write().unwrap() = String::from("production");

assert_eq!(configuration.read().unwrap().as_str(), "production");
```

Use `Arc<RwLock<T>>` when the lock itself needs shared ownership across threads.

## [`Cow<'a, T>`](https://doc.rust-lang.org/std/borrow/enum.Cow.html)

`Cow` means "clone on write." It can borrow data when no modification is needed and allocate an owned value only when a change is required.

```rust
use std::borrow::Cow;

fn ensure_question_mark(input: &str) -> Cow<'_, str> {
    if input.ends_with('?') {
        Cow::Borrowed(input)
    } else {
        Cow::Owned(format!("{input}?"))
    }
}

let unchanged = ensure_question_mark("Ready?");
let changed = ensure_question_mark("Ready");

assert!(matches!(unchanged, Cow::Borrowed(_)));
assert!(matches!(changed, Cow::Owned(_)));
```

`Cow` is useful in parsers, validation, normalization, and APIs that usually return input unchanged but occasionally need an owned modification.

## [`Pin<P>`](https://doc.rust-lang.org/std/pin/struct.Pin.html)

`Pin<P>` wraps a pointer `P` and restricts access that could move its pointee. This matters for address-sensitive values, including the state machines generated for many asynchronous futures and carefully designed self-referential types.

```rust
use std::pin::Pin;

fn inspect(value: Pin<&String>) {
    println!("{}", value.get_ref());
}

let value: Pin<Box<String>> = Box::pin(String::from("stable address"));
inspect(value.as_ref());
```

`Pin` does not automatically make every value immovable. Types that implement `Unpin`, including most ordinary Rust types, can still be moved safely. Pinning becomes significant when an API works with a `!Unpin` type and relies on its address remaining stable.

## Common combinations

| Type or combination | Main purpose | Borrow checking | Thread use |
| :-- | :-- | :-- | :-- |
| `Box<T>` | Exclusive heap ownership | Compile time | Can cross threads when `T` permits it |
| `Rc<T>` | Shared ownership | Compile time | Single-threaded only |
| `Arc<T>` | Shared ownership | Compile time | Thread-safe when `T` is `Send + Sync` |
| `Weak<T>` | Non-owning observation; break cycles | Compile time | `rc::Weak` is single-threaded; `sync::Weak` is thread-safe when `T` permits it |
| `Cell<T>` | Replace or copy small values through shared access | By-value API | Single-threaded only |
| `RefCell<T>` | Mutable borrowing through shared access | Runtime | Single-threaded only |
| `Rc<RefCell<T>>` | Shared mutable state | Runtime | Single-threaded only |
| `Arc<Mutex<T>>` | Shared mutable state with one accessor | Runtime locking | Multi-threaded |
| `Arc<RwLock<T>>` | Shared state with many readers or one writer | Runtime locking | Multi-threaded |
| `Cow<'a, T>` | Borrow first; clone only when needed | Compile time | Depends on `T` |
| `Pin<P>` | Prevent moving address-sensitive pointees through `P` | Compile time API restrictions | Depends on `P` and its pointee |

The combinations used most often are:

- `Box<dyn Trait>` for exclusive ownership with dynamic dispatch;
- `Rc<RefCell<T>>` for shared mutable state in one thread;
- `Arc<T>` for shared immutable state across threads;
- `Arc<Mutex<T>>` for shared mutable state across threads;
- `Arc<RwLock<T>>` for shared, read-heavy state across threads; and
- `Weak<T>` for non-owning links and cycle prevention.
