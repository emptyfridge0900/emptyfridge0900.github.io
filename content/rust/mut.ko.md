+++
title="Mut and reference"
date=2023-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

Rust에서 `mut`은 서로 다른 두 가지를 설명할 수 있다.

1. binding 자체를 다시 할당할 수 있는가?
2. reference를 통해 가리키는 값을 변경할 수 있는가?

아래 네 형태는 모두 다르다.

```rust
let     a:     &T;     // shared reference를 담는 immutable binding
let mut a:     &T;     // shared reference를 담는 mutable binding
let     a: &mut T;     // mutable reference를 담는 immutable binding
let mut a: &mut T;     // mutable reference를 담는 mutable binding
```

`let mut a: &T`는 `a`가 다른 shared reference를 가리키도록 재할당될 수 있다는 뜻이다. 하지만 `a`를 통해 `T` 값을 변경할 수는 없다.

`let a: &mut T`는 `a` 자체를 다른 reference로 재할당할 수는 없지만, `a`가 가리키는 값은 변경할 수 있다는 뜻이다.

function parameter도 같은 규칙을 따른다.

```rust
fn update<T>(input: &mut T) {
    // input은 개념적으로 아래 binding과 같은 모양이다:
    // let input: &mut T = ...
}
```

따라서 `input: &mut T`는 parameter binding인 `input` 자체가 mutable이라는 뜻이 아니다. `input`이 `T`에 대한 mutable reference라는 뜻이다.

함수가 reference 뒤의 값을 변경하기만 하면 `input: &mut T`로 충분하다.

```rust
fn append_world(input: &mut String) {
    input.push_str(" world");
}
```

caller는 보통 평범한 `String`을 소유하고 있다가, call site에서 mutable reference를 만든다.

```rust
let mut a = "hello".to_string(); // a: String
append_world(&mut a);            // &mut a: &mut String
```

`a` 자체는 reference가 아니다. `a`의 타입은 여전히 `String`이다. `&mut a`라는 expression이 `a`를 잠시 `&mut String`으로 빌려서 function parameter와 맞춘다.

parameter type과 맞아야 하는 것은 원래 변수의 type이 아니라, 함수에 넘기는 argument expression의 type이다.

`i32` parameter에서는 변수와 argument expression이 같은 경우가 많다.

```rust
fn use_number(input: i32) {
    println!("{input}");
}

let a = 5;       // a: i32
use_number(a);   // a: i32
```

반면 `&mut String` parameter에서는 argument expression을 변수에서 새로 만들어 넘기는 경우가 많다.

```rust
let mut a = "hello".to_string(); // a: String
append_world(&mut a);            // &mut a: &mut String
```

즉 함수가 받는 값의 type은 `&mut String`이 맞다. 하지만 caller의 변수 자체를 `&mut String`으로 선언해야 하는 것은 아니다. `&mut a` expression이 `&mut String` type을 만든다.

아래 코드는 compile되지 않는다.

```rust
let a = "hello".to_string();
append_world(&mut a); // error: cannot borrow `a` as mutable
```

문제는 `a`가 immutable binding이라는 점이다. `String`은 내부 값을 변경할 수 있는 타입이지만, Rust는 `&mut a`를 만들기 전에 변수 binding이 `mut`로 선언되어 있기를 요구한다.

`mut`은 type annotation의 일부가 아니므로 아래 syntax는 invalid다.

```rust
let a: mut = "hello".to_string(); // invalid Rust
```

`mut`은 `let` 뒤, 변수 이름 앞에 쓴다.

```rust
let mut a = "hello".to_string();
```

명시적 type annotation을 붙여도 순서는 같다.

```rust
let mut a: String = "hello".to_string();
```

binding의 type을 `&mut String`으로 쓰는 것도 가능하다. 다만 이 경우 binding은 `String`을 소유하는 것이 아니라 reference를 저장한다.

```rust
let mut s = "hello".to_string();
let input: &mut String = &mut s;

input.push_str(" world");
```

이것은 `append_world`가 parameter를 받을 때 일어나는 일과 개념적으로 가깝다.

```rust
let mut s = "hello".to_string();
append_world(&mut s);

// append_world 안에서 parameter는 대략 이런 binding처럼 볼 수 있다:
// let input: &mut String = &mut s;
```

함수 내부에서 local parameter variable 자체를 다른 reference로 재할당해야 한다면 binding에도 `mut`이 필요하다.

```rust
fn point_to_other<'a>(mut input: &'a mut String, other: &'a mut String) {
    input.push('!');

    input = other;
    input.push('!');
}
```

여기서 `mut input`은 local reference variable인 `input`을 재할당 가능하게 만들 뿐이다. caller의 변수가 다른 곳을 가리키게 만드는 것은 아니다.

예제에서 두 reference에 같은 lifetime `'a`를 쓰는 이유는 `input = other` 때문이다. `other`가 `input`이 기대하는 것과 같은 lifetime 동안 유효해야 한다. 일반 코드에서는 이 패턴이 흔하지 않다. 대부분의 함수는 reference 자체를 다시 묶는 것이 아니라 reference 뒤의 값을 변경하므로 `input: &mut T`만 있으면 충분하다.

## Immutable binding은 재할당할 수 없다

```rust
let a = "hello".to_string();
a = a + " world"; // error: cannot assign twice to immutable variable
```

`+` operator는 왼쪽 `String`을 consume하고 새 `String`을 return한다. 여기서 문제는 string 연산 자체가 아니다. `a`가 `mut`로 선언되지 않았는데 새 값을 다시 `a`에 assign하려고 한 것이 문제다.

binding을 다시 할당해야 한다면 `let mut`을 사용한다.

```rust
let mut a = "hello".to_string();
a = a + " world";
```

## Move와 borrow

```rust
let a = "hello".to_string();
let b = a;  // ownership이 a에서 b로 move된다
let c = &a; // error: 위에서 a가 move되었으므로 borrow할 수 없다
let d = &b; // ok: b를 borrow한다
```

`String`은 heap data를 소유하므로 `let b = a`는 ownership을 `a`에서 `b`로 move한다. move 이후 `a`는 더 이상 유효하지 않다.

## Mutable borrow

```rust
let mut a = "hello".to_string();
let b = &mut a; // mutable borrow, move가 아니다

b.push(' ');
b.push_str("world");
```

`b`는 `a`에 대한 mutable reference다. `String`의 ownership을 가져가지 않는다.

mutable borrow가 아직 사용 중이면 `a`를 직접 사용할 수 없다.

```rust
let mut a = "hello".to_string();
let b = &mut a;

// println!("{a}"); // b의 마지막 사용 전이라면 error
b.push_str(" world");
```

Rust는 동시에 여러 shared reference를 허용하거나 하나의 mutable reference를 허용하지만, 둘을 동시에 허용하지 않는다.

## Mutable binding 재할당

```rust
let mut a = "hello".to_string();
let b = "world".to_string();

a = b; // b가 a로 move된다
```

`a`가 mutable binding이므로 다시 assign할 수 있다. `b`의 값이 `a`로 move되고, `a`가 이전에 들고 있던 값은 drop된다.

이 assignment 이후 `b`는 더 이상 유효하지 않다.

## Mutable reference를 통해 assign하기

```rust
let mut a = "hello".to_string();
let r = &mut a;
let b = "world".to_string();

*r = b;
```

여기서 `r`을 `mut`로 선언할 필요는 없다. `r` 자체를 다른 reference로 재할당하는 것이 아니라, `*r`을 통해 mutable reference 뒤의 값에 assign하기 때문이다.

따라서 아래 구분이 중요하다.

```rust
let mut a = "hello".to_string();
let r = &mut a; // r은 다른 곳을 가리키도록 재할당될 수 없다
*r = "world".to_string(); // ok: r 뒤의 값을 변경한다
```

```rust
let mut first = "first".to_string();
let mut second = "second".to_string();

let mut r = &mut first; // r 자체를 재할당할 수 있다
r.push('!');

r = &mut second;
r.push('!');
```
