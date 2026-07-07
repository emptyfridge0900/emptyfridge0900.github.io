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
fn update<T>(input: &mut T) {
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

The caller usually owns a normal `String`, then creates the mutable reference at the call site:

```rust
let mut a = "hello".to_string(); // a: String
append_world(&mut a);            // &mut a: &mut String
```

`a` itself is not a reference. Its type is still `String`. The expression `&mut a` temporarily borrows `a` as `&mut String` so it matches the function parameter.

The parameter type must match the argument expression, not necessarily the original variable's type.

For an `i32` parameter, the variable and the argument expression are often the same thing:

```rust
fn use_number(input: i32) {
    println!("{input}");
}

let a = 5;       // a: i32
use_number(a);   // a: i32
```

For a `&mut String` parameter, the argument expression is usually created from the variable:

```rust
let mut a = "hello".to_string(); // a: String
append_world(&mut a);            // &mut a: &mut String
```

So the function receives a value whose type is `&mut String`, but the caller's variable does not have to be declared as `&mut String`. The expression `&mut a` is the part that has type `&mut String`.

This does not compile:

```rust
let a = "hello".to_string();
append_world(&mut a); // error: cannot borrow `a` as mutable
```

The problem is that `a` is an immutable binding. `String` is a type that can be mutated, but Rust still requires the variable binding to be declared with `mut` before you can create `&mut a`.

`mut` is not part of the type annotation, so this is invalid:

```rust
let a: mut = "hello".to_string(); // invalid Rust
```

Write `mut` after `let`, before the variable name:

```rust
let mut a = "hello".to_string();
```

With an explicit type annotation, the order is still the same:

```rust
let mut a: String = "hello".to_string();
```

If you write a binding whose type is `&mut String`, that is also valid, but it means the binding stores a reference instead of owning the `String`:

```rust
let mut s = "hello".to_string();
let input: &mut String = &mut s;

input.push_str(" world");
```

This is conceptually close to what happens when `append_world` receives its parameter:

```rust
let mut s = "hello".to_string();
append_world(&mut s);

// Inside append_world, the parameter is like:
// let input: &mut String = &mut s;
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
