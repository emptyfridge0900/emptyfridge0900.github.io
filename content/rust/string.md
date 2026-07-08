+++
title="String in Rust"
date=2021-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### [Slice](https://doc.rust-lang.org/book/ch04-03-slices.html)

A string slice is a reference to part of a `String`.

```rust
let s = String::from("hello world");

let hello = &s[0..5];//hello
let world = &s[6..11];//world
```

Here, `s` is an immutable string slice.

```rust
let s:&'static str = "Hello, world!";//data type is &'static str
let my_string:String = s.to_string()//data type is String
```

Slice examples:

```rust
let my_string = String::from("hello world");
&my_string[..]//string slice
let my_string_literal = "hello world";
my_string_literal//string slice
my_string_literal[..]//string slice
```

### [String](https://doc.rust-lang.org/book/ch08-02-strings.html)

Rust core supports only one string type: the string slice, `str`.

You usually see it in the form `&str`.

`String` is a type provided by Rust's standard library.

Rust's standard library also provides string-related types such as `OsString`, `OsStr`, `CString`, and `CStr`.

### Creating a new String

```rust
1.
let data = "initial contents";
let s = data.to_string();

2.
let s = "initial contents".to_string();

3.
let s = String::from("initial contents");
```

### Updating a String

```rust
1.
let mut s = String::from("foo");
s.push_str("bar");

2.
let mut s1 = String::from("foo");
let s2 = "bar";
s1.push_str(s2);

3.
let mut s = String::from("lo");
s.push('l');
```

### Concatenating Strings

In this example, `s1` can no longer be used afterward. Also, when creating `s3`, `s2` must have `&` in front of it so it becomes an `&String`.

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2;
```

Actually, `+` is replaced with a function called `add()`.

```rust
s1.add(s2)
```

It has this shape. Take a quick look at the signature of `add()`.

```rust
fn add(self, s: &str) -> String {
```

It receives an argument of type `s: &str`. Since `&String` is coerced to `&str`, we do not need to worry about it here.

### Indexing

The following code produces an error because Rust strings do not support indexing.

```rust
let s1 = String::from("hello");
let h = s1[0];
```

### UTF-8

```rust
let hello = String::from("Hola");// 4bytes
let hello = String::from("Здравствуйте");//12 characters but 24 bytes
let hello = "Здравствуйте";
for n in hello.as_bytes(){
    println!("{}",n);

}
```

This prints 6 chars.

```rust
for c in "नमस्ते".chars() {
    println!("{}", c);
}
```

This prints 18 bytes.

```rust
for b in "नमस्ते".bytes() {
    println!("{}", b);
}
```
