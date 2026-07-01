+++
title = "T: Send + Sync + 'static 완전히 이해하기"
date = 2026-07-01

[taxonomies]
categories = ["Post"]
tags = ["post", "Rust"]
+++

Rust에서 스레드나 비동기 작업을 다루다 보면 다음 제약을 자주 만난다.

```rust
T: Send + Sync + 'static
```

처음 보면 `T`를 프로그램이 종료될 때까지 유지하면서 다른 스레드로 보내라는 뜻처럼 보인다. 하지만 실제 의미는 다르다.

먼저 한 문장으로 정리하면 다음과 같다.

> `T: Send + Sync + 'static`은 `T`의 소유권을 다른 스레드로 안전하게 옮길 수 있고, 여러 스레드가 `T`를 공유 참조로 안전하게 접근할 수 있으며, `T` 안에 수명이 짧은 참조가 없다는 뜻이다.

`Send`, `Sync`, `'static`은 서로 독립된 조건이다. 하나씩 분리해서 살펴보자.

## `Send`: 다른 스레드로 소유권을 옮겨도 안전하다

`Send`는 값을 한 스레드에서 다른 스레드로 안전하게 옮길 수 있음을 나타내는 마커 트레이트다.

```rust
pub unsafe auto trait Send {}
```

대부분의 일반적인 타입은 구성 요소가 모두 `Send`라면 자동으로 `Send`가 된다. 예를 들어 `String`, `Vec<T>`, `Box<T>`는 내부 타입이 조건을 만족하면 다른 스레드로 이동할 수 있다.

반면 `Rc<T>`는 `Send`가 아니다. 참조 카운트를 원자적으로 갱신하지 않기 때문에 여러 스레드에서 사용하면 안전하지 않다.

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

이 코드는 컴파일되지 않는다. `move` 클로저가 `Rc<String>`의 소유권을 새 스레드로 옮기려고 하지만 `Rc<String>: Send`가 아니기 때문이다.

## `Sync`: 여러 스레드가 공유 참조로 접근해도 안전하다

`Sync`는 여러 스레드가 하나의 값에 대한 공유 참조를 안전하게 사용할 수 있음을 나타내는 마커 트레이트다.

```rust
pub unsafe auto trait Sync {}
```

Rust 표준 라이브러리는 그 관계를 다음과 같이 정의한다.

> `T`가 `Sync`라는 것은 `&T`가 `Send`라는 것과 같다.

즉, `&T`를 다른 스레드로 보내도 안전하다면 `T: Sync`다. `Sync`는 “소유권을 이동해도 되는가?”가 아니라 “공유 참조를 여러 스레드에서 사용해도 되는가?”에 답한다.

`Cell<T>`는 두 조건의 차이를 잘 보여준다. `Cell<i32>`는 소유권 전체를 다른 스레드로 옮길 수 있으므로 `Send`지만, 공유 참조를 통해 내부 값을 변경할 수 있고 그 변경을 동기화하지 않으므로 `Sync`는 아니다.

```rust
use std::cell::Cell;

fn require_send<T: Send>() {}
fn require_sync<T: Sync>() {}

fn main() {
    require_send::<Cell<i32>>();
    require_sync::<Cell<i32>>(); // 컴파일 오류: Cell<i32>는 Sync가 아니다.
}
```

여러 스레드에서 값을 공유해야 한다면 보통 `Arc<T>`를 사용한다.

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

단, `Arc<T>`라고 해서 내부 값이 자동으로 스레드 안전해지는 것은 아니다. `Arc`는 참조 카운트만 원자적으로 관리한다. `Arc<T>`를 스레드 사이에서 안전하게 이동하려면 내부의 `T`도 `Send + Sync` 조건을 만족해야 한다.

공유하면서 값을 변경해야 한다면 `Arc<Mutex<T>>`, `Arc<RwLock<T>>`, 원자 타입처럼 접근 자체를 동기화하는 도구가 추가로 필요하다.

다음처럼 공유 값을 백그라운드 스레드로 보내는 제네릭 API에서 세 조건이 함께 등장할 수 있다.

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

- `Send`: `Arc<T>`의 소유권을 새 스레드로 옮길 수 있어야 한다.
- `Sync`: 여러 `Arc`가 같은 `T`를 공유 참조로 접근할 수 있어야 한다.
- `'static`: `T`가 호출자의 스택에 있는 수명이 짧은 참조를 포함해서는 안 된다.

## `'static`: 값이 영원히 살아야 한다는 뜻이 아니다

`'static`은 두 위치에서 서로 다르게 보인다.

첫 번째는 참조 자체의 수명이다.

```rust
let message: &'static str = "hello";
```

문자열 리터럴은 프로그램 바이너리에 저장되므로 이 참조는 프로그램 전체 실행 시간 동안 유효하다.

두 번째는 타입에 붙는 수명 제약이다.

```rust
T: 'static
```

이 표현은 `T`가 프로그램 종료까지 반드시 살아 있어야 한다는 뜻이 아니다. `T` 안의 모든 참조가 `'static`보다 짧지 않아야 한다는 뜻이다. 더 실용적으로 말하면, `T`가 수명이 짧은 빌린 값을 포함하지 않아야 한다.

소유한 값은 보통 이 조건을 자연스럽게 만족한다.

```rust
fn accept_static<T: 'static>(value: T) {
    drop(value);
}

fn main() {
    let message = String::from("hello");
    accept_static(message);
}
```

`String`은 자신의 데이터를 소유하므로 `String: 'static`이다. 하지만 위 함수는 값을 즉시 버린다. 따라서 `T: 'static`이 값 자체를 영원히 살게 한다는 해석은 틀렸다.

다음 타입은 수명이 짧은 참조를 포함하므로 `'static` 조건을 만족하지 못한다.

```rust
fn accept_static<T: 'static>(_: T) {}

fn main() {
    let message = String::from("hello");
    let borrowed = &message;

    accept_static(borrowed);
}
```

`borrowed`의 타입은 지역 변수 `message`의 수명에 묶인 `&String`이다. 이 참조는 `'static`이 아니다.

## 왜 `thread::spawn`에는 `Send + 'static`만 보일까?

`std::thread::spawn`의 시그니처를 단순화하면 다음과 같다.

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

새 스레드는 `spawn`을 호출한 함수보다 오래 실행될 수 있다. `JoinHandle`을 버리면 스레드는 분리되어 계속 실행될 수도 있다. 컴파일러는 새 스레드가 언제 종료될지 알 수 없으므로 클로저가 지역 변수의 짧은 참조를 들고 떠나는 것을 허용할 수 없다. 이것이 `'static`이 필요한 이유다.

클로저 자체는 새 스레드로 이동한다. 따라서 클로저가 캡처한 값도 안전하게 이동할 수 있어야 한다. 이것이 `Send`가 필요한 이유다.

반환값 `T`에도 같은 제약이 붙는다. `T`는 새 스레드에서 생성되어 `JoinHandle<T>`에 보관된 뒤 `join`을 호출한 스레드로 전달되므로 `Send`여야 한다. 일반 스레드와 `JoinHandle<T>`은 생성된 스코프 밖으로 이동할 수 있으므로, `T: 'static`은 반환값이 해당 스코프에 묶인 짧은 참조를 포함하지 않도록 보장한다.

이 제약이 새 스레드 자신의 지역 변수를 가리키는 참조의 반환을 막는 것은 아니다. 그런 참조는 함수가 끝나면 무효가 되므로 `'static` 제약과 관계없이 빌림 검사기가 금지한다. 반면 `ScopedJoinHandle<'scope, T>`은 스코프 밖으로 나갈 수 없으므로 `T: 'scope`만 요구하며, 바깥 스코프에서 빌린 값을 결과로 반환할 수 있다.

여기에는 `Sync`가 없다. 값을 새 스레드 하나로 **이동**하기만 한다면 여러 스레드가 동시에 공유 참조로 접근하지 않기 때문이다. 예를 들어 `Cell<i32>`는 `Sync`가 아니지만 소유권 전체를 새 스레드로 옮기는 다음 코드는 유효하다.

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

따라서 스레드를 사용한다는 이유만으로 모든 제네릭 타입에 `Sync`를 추가하면 안 된다. 실제로 여러 스레드가 같은 값을 공유할 때만 필요하다.

## `move`는 빌린 값을 `'static`으로 만들지 않는다

다음 코드는 흔히 만나는 실패 사례다.

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

`move`는 클로저가 캡처한 **변수**를 값으로 가져가게 한다. 하지만 여기서 가져가는 값은 `String`이 아니라 `&String`이다. 참조를 이동해도 그 참조가 가리키는 데이터의 수명은 늘어나지 않는다.

소유한 `String` 자체를 클로저로 옮기면 해결된다.

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

이제 클로저가 데이터를 직접 소유한다. `String`은 `Send + 'static`을 만족하므로 새 스레드가 얼마나 오래 실행되든 참조가 무효해질 위험이 없다.

## 빌린 값을 사용해야 한다면 `thread::scope`

항상 데이터를 복제하거나 `Arc`로 감쌀 필요는 없다. 스레드가 특정 범위 안에서 반드시 종료된다는 사실을 보장할 수 있다면 `std::thread::scope`를 사용할 수 있다.

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

`thread::scope`는 범위를 벗어나기 전에 내부에서 생성한 모든 스레드를 조인한다. 따라서 새 스레드가 `message`보다 오래 살 수 없고, 클로저가 non-`'static` 참조를 빌릴 수 있다.

여기서도 스레드 경계는 존재하므로 안전한 스레드 간 이동에 필요한 `Send` 조건까지 사라지는 것은 아니다. 사라지는 것은 독립적으로 실행되는 스레드에 필요했던 `'static` 제약이다.

## 세 조건은 서로 독립적이다

다음처럼 생각하면 구분하기 쉽다.

| 타입 | `Send` | `Sync` | `'static` | 이유 |
|---|---:|---:|---:|---|
| `String` | 예 | 예 | 예 | 데이터를 소유하며 안전하게 이동하거나 공유할 수 있다. |
| `Rc<String>` | 아니요 | 아니요 | 예 | 데이터를 소유하지만 참조 카운트가 스레드 안전하지 않다. |
| `Cell<i32>` | 예 | 아니요 | 예 | 소유권은 이동할 수 있지만 공유 참조를 통한 변경은 동기화되지 않는다. |
| 지역 `String`을 가리키는 `&String` | 예 | 예 | 아니요 | 참조 대상은 공유해도 안전하지만 지역 값의 수명에 묶여 있다. |
| `&'static str` | 예 | 예 | 예 | 참조 대상이 프로그램 전체 실행 시간 동안 유효하다. |

즉, 소유한 타입이라고 해서 반드시 `Send`나 `Sync`인 것은 아니며, 스레드 안전한 참조라고 해서 반드시 `'static`인 것도 아니다.

## 비동기 코드에서도 같은 원리다

멀티 스레드 런타임에서 사용하는 `tokio::spawn`도 일반적으로 future에 `Send + 'static`을 요구한다.

- `Send`: 런타임이 future를 다른 워커 스레드로 옮길 수 있다.
- `'static`: 생성된 task가 현재 함수보다 오래 실행될 수 있다.

`spawn_local`은 task를 현재 로컬 실행기에서만 실행하므로 `Send`를 요구하지 않는다. 하지만 task가 호출자의 스택을 자유롭게 빌릴 수 있다는 뜻은 아니므로 일반적으로 `'static` 제약은 여전히 남는다.

`tokio::spawn`도 future 자체에 `Sync`를 요구하지는 않는다. 다만 future가 `&T`를 잡은 채 `.await` 지점을 넘어가고 런타임이 그 future를 다른 스레드로 옮길 수 있다면, 그 참조가 `Send`이기 위해 `T: Sync`가 간접적으로 필요해질 수 있다.

## 언제 직접 추가해야 할까?

제네릭 타입에 습관적으로 `Send + Sync + 'static`을 추가하면 API가 불필요하게 제한된다. 실제 동작이 요구할 때만 추가하는 것이 좋다.

`Send`가 필요한 대표적인 경우는 다음과 같다.

- 값을 다른 OS 스레드로 이동한다.
- 멀티 스레드 비동기 실행기가 작업을 워커 스레드 사이에서 이동할 수 있다.
- 값을 스레드 안전한 큐나 채널을 통해 전달한다.

`Sync`가 필요한 대표적인 경우는 다음과 같다.

- 여러 스레드가 `&T`를 동시에 사용한다.
- `Arc<T>`로 같은 값을 여러 스레드나 task가 공유한다.
- 서버 상태, 핸들러, 콜백처럼 하나의 인스턴스를 동시 호출자들이 공유한다.

`'static`이 필요한 대표적인 경우는 다음과 같다.

- 작업이 현재 함수 호출보다 오래 유지될 수 있다.
- 값을 장기 보관하는 객체나 전역 레지스트리에 저장한다.
- 콜백이나 task를 나중에 실행하기 위해 저장한다.

`Send + 'static`이 필요한 경우는 “호출자의 수명과 독립적으로 실행될 작업을 다른 스레드로 보낸다”라고 이해하면 된다. 여기에 `Sync`까지 붙는다면 그 값을 여러 스레드가 공유 참조로 접근할 수 있다는 계약도 추가된다.

```rust
use std::thread::{self, JoinHandle};

fn run_in_background<T>(value: T) -> JoinHandle<()>
where
    T: Send + 'static,
{
    thread::spawn(move || drop(value))
}
```

반대로 함수가 값을 동기적으로 사용하고 반환한다면 대개 이런 제약은 필요 없다. 짧은 참조를 받으면서도 안전하게 작업할 수 있기 때문이다.

## 컴파일 오류를 만났을 때의 판단 순서

`Send + Sync + 'static` 관련 오류가 나오면 무조건 제약을 추가하기보다 다음 순서로 확인하자.

1. **작업이 정말 호출자보다 오래 살아야 하는가?** 그렇지 않다면 직접 실행하거나 `thread::scope`처럼 수명이 제한된 API를 고려한다.
2. **클로저가 참조 대신 소유한 값을 가져갈 수 있는가?** `move`와 함께 값을 이동하거나 필요한 부분만 복제한다.
3. **여러 작업이 데이터를 공유해야 하는가?** 필요하다면 `Arc`, `Mutex`, 채널 같은 동시성 도구를 고려한다.
4. **타입이 왜 `Send`가 아닌가?** `Rc`, 스레드 로컬 값, 락 가드처럼 스레드 경계를 넘으면 안 되는 값이 포함되어 있는지 확인한다.
5. **타입이 정말 `Sync`여야 하는가?** 소유권을 한 스레드로 이동할 뿐이라면 `Sync`는 필요하지 않을 수 있다. 공유가 필요하다면 동기화되지 않은 내부 가변성이 있는지 확인한다.
6. **정말 멀티 스레드 실행이 필요한가?** 단일 스레드 로컬 실행기를 사용할 수 있다면 `Send`와 `Sync` 요구를 피할 수 있다.

`T: Send + Sync + 'static`은 컴파일러를 달래기 위한 주문이 아니다. API가 값의 소유권을 **어디로 이동시키는지**, 값을 **어떻게 공유하는지**, 그리고 **얼마나 오래 보관할 수 있는지**를 호출자에게 알리는 계약이다.

## 참고 자료

- [Rust 표준 라이브러리: `Send`](https://doc.rust-lang.org/std/marker/trait.Send.html)
- [Rust 표준 라이브러리: `Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html)
- [Rust 표준 라이브러리: `std::thread::spawn`](https://doc.rust-lang.org/std/thread/fn.spawn.html)
- [Rust 표준 라이브러리: `std::thread::scope`](https://doc.rust-lang.org/std/thread/fn.scope.html)
- [The Rust Reference: Lifetime bounds](https://doc.rust-lang.org/reference/trait-bounds.html#lifetime-bounds)
- [Tokio: `tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html)
