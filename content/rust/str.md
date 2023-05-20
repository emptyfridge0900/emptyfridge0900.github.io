+++
title="String and str "
date=2023-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### <code>String</code>와 <code>str</code>차이
<code>String</code>은 <code>Vec<T></code>이랑 비슷하다. C++의 String class와 비슷.
<code>str</code>은 <code>char[]</code>랑 비슷하다. C++에서 char array를 사용할때 변수명을 통해 pointer를 사용하는 것과 같이, &를 str앞에 붙여서 레퍼런스로 사용한다.

### 타입변환
<code>String</code> 타입을 <code>&str</code>으로 변환하려면?
&*을 String 앞에 붙인다. [Deref](https://doc.rust-lang.org/std/string/struct.String.html#deref) 참조

```rust
let s = String::from("hello world")
let ss = *s; //str type, Deref coerce 
let sss = &*s; //&str
```
또는
as_str()함수를 사용한다

```rust
let s = String::from("hello world")
let ss = s.as_str; //&str
```

<code>&'static str</code> 타입(string static)을 <coe>String</code>으로 변환하려면?
to_string()함수를 사용하면된다.
<code>&str</code>을 <code>String</code>으로 변환하는 것은 메모리를 할당해야하기 때문에 될수 있으면 하지말자

```rust
let s = "hello world";
let ss = s.to_string();
```

### Byte String Literal
b"whatever"은 byte string literal이다. 타입은 buf:&[u8;8] 이다.

```rust
let s = b"hello world"; //buf:&[u8;11]
let ss =  b"whatever"; //buf:&[u8;8]
```