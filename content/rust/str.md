+++
title="String and str"
date=2023-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### Difference between <code>String</code> and <code>str</code>

<code>String</code> is similar to <code>Vec&lt;T&gt;</code>. It is similar to the C++ `String` class.

<code>str</code> is similar to <code>char[]</code>. Just as C++ uses a variable name as a pointer when working with a char array, Rust usually uses `str` through a reference by putting `&` in front of it.

### Type conversion

How do you convert a <code>String</code> into <code>&str</code>?

Put `&*` in front of the `String`. See [Deref](https://doc.rust-lang.org/std/string/struct.String.html#deref).

```rust
let s = String::from("hello world")
let ss = *s; //str type, Deref coerce
let sss = &*s; //&str
```

Or use the `as_str()` function.

```rust
let s = String::from("hello world")
let ss = s.as_str; //&str
```

How do you convert <code>&'static str</code> into <code>String</code>?

Use `to_string()`.

Converting <code>&str</code> into <code>String</code> requires allocation, so avoid it when possible.

```rust
let s = "hello world";
let ss = s.to_string();
```

### Byte String Literal

`b"whatever"` is a byte string literal. Its type is `buf: &[u8; 8]`.

```rust
let s = b"hello world"; //buf:&[u8;11]
let ss =  b"whatever"; //buf:&[u8;8]
```
