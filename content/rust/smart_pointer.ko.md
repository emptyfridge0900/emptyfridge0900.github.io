+++
title="Smart Pointer"
date=2025-05-24

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

Rust에는 단일한 `SmartPointer` trait가 없다. 대신 smart pointer라는 말은 값을 소유하거나 감싸면서 pointer처럼 동작하는 타입들을 가리키는 느슨한 표현이다. 보통 [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html)와 [`Drop`](https://doc.rust-lang.org/std/ops/trait.Drop.html)을 통해 그런 동작을 제공한다.

아래에 있는 `RefCell<T>`, `Mutex<T>`, `Pin<P>` 같은 타입은 엄밀히 말하면 interior mutability type이나 pointer wrapper에 더 가깝다. 그래도 실제 Rust 코드에서 owning smart pointer와 자주 조합되기 때문에 함께 정리한다.

## [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html)

`Box<T>`는 heap에 저장된 값 하나를 소유한다. 다음과 같은 경우에 사용한다.

- recursive type이나 trait object처럼 compile time에 크기를 알 수 없는 타입을 다룰 때
- 큰 값을 이동하되 heap allocation 자체는 복사하고 싶지 않을 때
- 어떤 component가 heap value를 exclusive하게 소유해야 할 때

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));
```

`Box`는 recursive field에 알려진 크기를 부여한다. 다른 `List`를 직접 embedding하는 대신 fixed-size pointer를 저장하기 때문이다.

## [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html)

`Rc<T>`는 single thread 안에서 reference counting으로 shared ownership을 제공한다. `Rc`를 clone하면 내부 값이 복사되는 것이 아니라 reference count만 증가한다.

```rust
use std::rc::Rc;

let name = Rc::new(String::from("Ferris"));
let first_owner = Rc::clone(&name);
let second_owner = Rc::clone(&name);

assert_eq!(first_owner.as_str(), "Ferris");
assert_eq!(second_owner.as_str(), "Ferris");
assert_eq!(Rc::strong_count(&name), 3);
```

syntax tree, GUI tree, graph node처럼 single-threaded structure에서 read-only data를 공유할 때 `Rc<T>`를 쓴다. `Rc<T>`는 `Send`도 `Sync`도 아니다. thread boundary를 넘어 ownership을 공유해야 한다면 `Arc<T>`를 사용한다.

## [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html)

`Arc<T>`는 `Rc<T>`의 atomic, thread-safe counterpart다. atomic reference counting에는 작은 synchronization cost가 있으므로 thread가 필요 없다면 `Rc<T>`를 선호한다.

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

`Arc<T>`가 공유하는 것은 ownership이지 mutability가 아니다. shared state를 변경해야 한다면 보통 `Mutex<T>`나 `RwLock<T>`와 함께 사용한다.

## [`RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html)

`RefCell<T>`는 single thread 안에서 interior mutability를 제공한다. Rust의 borrowing rule을 compile time이 아니라 runtime에 검사한다. 하나의 mutable borrow 또는 여러 immutable borrow는 가능하지만 둘을 동시에 가질 수 없다. 이 규칙을 어기면 panic이 발생한다.

```rust
use std::cell::RefCell;

let message = RefCell::new(String::from("Hello"));

message.borrow_mut().push_str(", Rust");

assert_eq!(message.borrow().as_str(), "Hello, Rust");
```

자주 쓰는 `Rc<RefCell<T>>` 조합은 single thread 안에서 여러 owner가 같은 값에 mutable access할 수 있게 해 준다. runtime borrow conflict를 피하려면 `borrow()`나 `borrow_mut()` scope를 짧게 유지하는 것이 좋다.

## [`Weak<T>`](https://doc.rust-lang.org/std/rc/struct.Weak.html)

`Weak<T>`는 `Rc<T>`가 관리하는 allocation에 대한 non-owning reference다. [`std::sync::Weak<T>`](https://doc.rust-lang.org/std/sync/struct.Weak.html)는 `Arc<T>`와 함께 쓰는 버전이다.

weak reference는 값을 살아 있게 유지하지 않는다. `upgrade()`를 호출하면 strong owner가 아직 있을 때는 `Some(Rc<T>)`를 반환하고, 값이 drop된 뒤에는 `None`을 반환한다.

```rust
use std::rc::Rc;

let owner = Rc::new(String::from("cached value"));
let observer = Rc::downgrade(&owner);

assert_eq!(observer.upgrade().unwrap().as_str(), "cached value");

drop(owner);
assert!(observer.upgrade().is_none());
```

`Weak<T>`는 back-reference, cache, observer, parent pointer에 사용한다. 또한 `Rc`나 `Arc` reference cycle을 끊어 memory leak을 막는 표준적인 방법이다.

## [`Cell<T>`](https://doc.rust-lang.org/std/cell/struct.Cell.html)

`Cell<T>`는 가벼운 single-threaded interior mutability 타입이다. `RefCell<T>`와 달리 내부 값에 대한 reference를 반환하지 않는다. `T: Copy`이면 `get()`으로 값을 복사해 꺼내고, 아니면 값을 통째로 교체하는 방식으로 사용한다.

```rust
use std::cell::Cell;

let request_count = Cell::new(0_u32);

request_count.set(request_count.get() + 1);
request_count.set(request_count.get() + 1);

assert_eq!(request_count.get(), 2);
```

counter, flag, enum state처럼 작은 `Copy` 값에 적합하다. 더 크거나 `Copy`가 아닌 값에 borrowed access가 필요하면 `RefCell<T>`를 사용한다.

## [`Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html)

`Mutex<T>`는 한 번에 하나의 thread만 mutable access할 수 있게 한다. lock을 잡으면 guard가 반환되고, guard가 drop될 때 자동으로 unlock된다. shared ownership은 보통 `Arc`로 감싸서 만든다.

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

느린 작업이나 `.await` 지점까지 mutex guard를 들고 가지 않는 것이 좋다. async code에서 guard가 `.await`를 넘어 살아야 한다면 async runtime이 제공하는 synchronization primitive를 사용한다.

## [`RwLock<T>`](https://doc.rust-lang.org/std/sync/struct.RwLock.html)

`RwLock<T>`는 여러 reader 또는 하나의 writer를 허용한다. read-heavy workload에서는 concurrency를 높일 수 있지만, `Mutex<T>`보다 overhead가 크고 scheduling policy는 operating system에 따라 달라진다.

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

lock 자체를 여러 thread가 shared ownership으로 들고 있어야 한다면 `Arc<RwLock<T>>`를 사용한다.

## [`Cow<'a, T>`](https://doc.rust-lang.org/std/borrow/enum.Cow.html)

`Cow`는 "clone on write"를 뜻한다. 수정이 필요 없을 때는 data를 borrow하고, 변경이 필요할 때만 owned value를 allocate한다.

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

`Cow`는 parser, validation, normalization, 그리고 대부분 입력을 그대로 반환하지만 가끔 owned modification이 필요한 API에서 유용하다.

## [`Pin<P>`](https://doc.rust-lang.org/std/pin/struct.Pin.html)

`Pin<P>`는 pointer `P`를 감싸고, 그 pointer가 가리키는 값을 move할 수 있는 access를 제한한다. address-sensitive value에 중요하다. 예를 들면 많은 async future를 위해 생성되는 state machine이나, 조심스럽게 설계된 self-referential type이 있다.

```rust
use std::pin::Pin;

fn inspect(value: Pin<&String>) {
    println!("{}", value.get_ref());
}

let value: Pin<Box<String>> = Box::pin(String::from("stable address"));
inspect(value.as_ref());
```

`Pin`이 모든 값을 자동으로 immovable하게 만드는 것은 아니다. 대부분의 일반 Rust 타입처럼 `Unpin`을 구현하는 타입은 여전히 안전하게 move될 수 있다. pinning이 중요해지는 것은 API가 `!Unpin` 타입과 함께 동작하고, 그 값의 address가 안정적으로 유지된다는 사실에 의존할 때다.

## 자주 쓰는 조합

| Type or combination | 주 목적 | Borrow checking | Thread 사용 |
| :-- | :-- | :-- | :-- |
| `Box<T>` | heap value의 exclusive ownership | Compile time | `T`가 허용하면 thread 경계를 넘을 수 있음 |
| `Rc<T>` | shared ownership | Compile time | single-threaded only |
| `Arc<T>` | shared ownership | Compile time | `T: Send + Sync`이면 thread-safe |
| `Weak<T>` | non-owning observation; cycle 끊기 | Compile time | `rc::Weak`는 single-threaded, `sync::Weak`는 `T`가 허용하면 thread-safe |
| `Cell<T>` | shared access로 작은 값 copy/replace | By-value API | single-threaded only |
| `RefCell<T>` | shared access로 mutable borrow | Runtime | single-threaded only |
| `Rc<RefCell<T>>` | shared mutable state | Runtime | single-threaded only |
| `Arc<Mutex<T>>` | 한 accessor만 접근하는 shared mutable state | Runtime locking | multi-threaded |
| `Arc<RwLock<T>>` | 많은 reader 또는 하나의 writer가 접근하는 shared state | Runtime locking | multi-threaded |
| `Cow<'a, T>` | 먼저 borrow하고 필요할 때만 clone | Compile time | `T`에 따라 다름 |
| `Pin<P>` | address-sensitive pointee가 `P`를 통해 move되지 않게 제한 | Compile time API restriction | `P`와 pointee에 따라 다름 |

가장 자주 쓰는 조합은 다음과 같다.

- dynamic dispatch가 필요한 exclusive ownership에는 `Box<dyn Trait>`
- single thread 안의 shared mutable state에는 `Rc<RefCell<T>>`
- thread 사이에서 공유하는 immutable state에는 `Arc<T>`
- thread 사이에서 공유하는 mutable state에는 `Arc<Mutex<T>>`
- read-heavy shared state에는 `Arc<RwLock<T>>`
- non-owning link와 cycle 방지에는 `Weak<T>`
