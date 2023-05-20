+++
title="Mut and reference"
date=2023-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

```rust
let     a:     &T
let mut a:     &T
let     a: &mut T
let mut a: &mut T
```

```rust
let a = "hello".to_string();
a = a.add(" world");// this will throw error, cannot assign twice to immutalbe variable
let b = a;//move
let c = &a;// a is moved above line, cannot be borrowed
let d = &b;//borrow
```
```rust
let mut a = "hello".to_string();
let b = &mut a; //a is borrowed as MUTABLE, not moved
b.push(' ');
b.push_str("world");
```
```rust
let mut a = "hello".to_string();
let b = "world".to_string();
a=b; // because a is mut can be assigned twice, b value is moved to a
```
```rust
let a = &mut "hello".to_string();
let b = "world".to_string();
*a = b;
```