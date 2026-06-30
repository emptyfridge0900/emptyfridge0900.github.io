+++
title="Mut and reference"
date=2023-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

In Rust, `mut` can describe two different things:

1. Whether a binding can be reassigned.
2. Whether a reference allows mutation of the value it points to.

These four forms are different:

```rust
let     a:     &T;     // immutable binding to a shared reference
let mut a:     &T;     // mutable binding to a shared reference
let     a: &mut T;     // immutable binding to a mutable reference
let mut a: &mut T;     // mutable binding to a mutable reference
```

`let mut a: &T` means `a` can be reassigned to point to another shared reference, but you still cannot mutate the `T` through `a`.

`let a: &mut T` means `a` itself cannot be reassigned, but the value behind the reference can be mutated through `a`.

Function parameters follow the same rule:

```rust
fn update(input: &mut T) {
    // input has the same shape as:
    // let input: &mut T = ...
}
```

So `input: &mut T` means the parameter binding `input` is not mutable, but it is a mutable reference to `T`.

If the function needs to mutate the value behind the reference, `input: &mut T` is enough:

```rust
fn append_world(input: &mut String) {
    input.push_str(" world");
}
```

If the function needs to reassign the local parameter variable itself, then the binding must be `mut`:

```rust
fn point_to_other<'a>(mut input: &'a mut String, other: &'a mut String) {
    input.push('!');

    input = other;
    input.push('!');
}
```

That `mut input` only makes the local reference variable reassignable. It does not make the caller's variable point somewhere else.

The example uses the same lifetime `'a` for both references because `input = other` means `other` must be valid for the same lifetime that `input` expects. In normal code, this pattern is uncommon. Most functions only need `input: &mut T`, because they mutate the value behind the reference instead of rebinding the reference parameter.

## Immutable binding cannot be reassigned

```rust
let a = "hello".to_string();
a = a + " world"; // error: cannot assign twice to immutable variable
```

The `+` operator consumes the left-hand `String` and returns a new `String`. The problem here is not the string operation itself. The problem is assigning the new value back into `a`, because `a` was not declared as `mut`.

Use `let mut` if the binding should be reassigned:

```rust
let mut a = "hello".to_string();
a = a + " world";
```

## Move and borrow

```rust
let a = "hello".to_string();
let b = a;  // move ownership from a to b
let c = &a; // error: a was moved above, so it cannot be borrowed
let d = &b; // ok: borrow b
```

`String` owns heap data, so `let b = a` moves ownership from `a` to `b`. After the move, `a` is no longer valid.

## Mutable borrow

```rust
let mut a = "hello".to_string();
let b = &mut a; // mutable borrow, not a move

b.push(' ');
b.push_str("world");
```

`b` is a mutable reference to `a`. It does not take ownership of the `String`.

While the mutable borrow is still being used, `a` cannot be used directly:

```rust
let mut a = "hello".to_string();
let b = &mut a;

// println!("{a}"); // error if used before the last use of b
b.push_str(" world");
```

This is because Rust allows either many shared references or one mutable reference, but not both at the same time.

## Reassigning a mutable binding

```rust
let mut a = "hello".to_string();
let b = "world".to_string();

a = b; // b is moved into a
```

Because `a` is a mutable binding, it can be assigned again. The value of `b` moves into `a`, and the previous value of `a` is dropped.

After this assignment, `b` is no longer valid.

## Assigning through a mutable reference

```rust
let mut a = "hello".to_string();
let r = &mut a;
let b = "world".to_string();

*r = b;
```

`r` does not need to be declared as `mut` here because we are not reassigning `r` itself. We are assigning to the value behind the mutable reference with `*r`.

So this distinction matters:

```rust
let mut a = "hello".to_string();
let r = &mut a; // r cannot point somewhere else
*r = "world".to_string(); // ok: mutate the value behind r
```

```rust
let mut first = "first".to_string();
let mut second = "second".to_string();

let mut r = &mut first; // r itself can be reassigned
r.push('!');

r = &mut second;
r.push('!');
```
