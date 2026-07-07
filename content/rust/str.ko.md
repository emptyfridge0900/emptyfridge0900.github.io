+++
title="String and str"
date=2023-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### <code>String</code>과 <code>str</code>의 차이
<code>String</code>은 <code>Vec<T></code>와 비슷하게 heap에 저장되는, growable한 문자열 타입이다. C++의 `std::string`과 비슷하다고 볼 수 있다.
<code>str</code>은 문자열 데이터의 slice 타입이다. 크기를 compile time에 알 수 없는 unsized type이므로 보통 단독으로 쓰지 않고 <code>&str</code>처럼 reference로 사용한다.

### 타입변환
<code>String</code> 타입을 <code>&str</code>으로 변환하려면?
`String` 앞에 `&*`를 붙일 수 있다. [Deref](https://doc.rust-lang.org/std/string/struct.String.html#deref) 참고.

```rust
let s = String::from("hello world")
let ss = *s; //str type, Deref coerce
let sss = &*s; //&str
```
또는 `as_str()` 함수를 사용한다.

```rust
let s = String::from("hello world")
let ss = s.as_str; //&str
```

<code>&'static str</code> 타입(string literal)을 <code>String</code>으로 변환하려면?
`to_string()` 함수를 사용하면 된다.
<code>&str</code>을 <code>String</code>으로 변환하면 새 메모리 할당이 필요하므로, 꼭 소유한 문자열이 필요한 경우에만 하자.

```rust
let s = "hello world";
let ss = s.to_string();
```

### Byte String Literal
`b"whatever"`은 byte string literal이다. 타입은 `&[u8; 8]`처럼 byte array reference가 된다.

```rust
let s = b"hello world"; //buf:&[u8;11]
let ss =  b"whatever"; //buf:&[u8;8]
```
