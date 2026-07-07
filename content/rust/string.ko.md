+++
title="String in Rust"
date=2021-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### [Slice](https://doc.rust-lang.org/book/ch04-03-slices.html)
string slice는 `String`의 일부를 가리키는 reference다.

```rust
let s = String::from("hello world");

let hello = &s[0..5];//hello
let world = &s[6..11];//world
```

여기서 `s`는 immutable string slice다.
```rust
let s:&'static str = "Hello, world!";//data type is &'static str
let my_string:String = s.to_string()//data type is String
```

slice 예제:
```rust
let my_string = String::from("hello world");
&my_string[..]//string slice
let my_string_literal = "hello world";
my_string_literal//string slice
my_string_literal[..]//string slice
```

### [String](https://doc.rust-lang.org/book/ch08-02-strings.html)
Rust core language에는 문자열 타입이 하나만 있다. 바로 string slice인 `str`이다.
보통은 `&str` 형태로 자주 본다.

`String` 타입은 Rust standard library가 제공하는 growable, owned UTF-8 string 타입이다.
standard library에는 `String` 외에도 `OsString`, `OsStr`, `CString`, `CStr` 같은 문자열 관련 타입이 있다.

### 새로운 String 만들기
```rust
1.
let data = "initial contents";
let s = data.to_string();

2.
let s = "initial contents".to_string();

3.
let s = String::from("initial contents");
```

### String 업데이트하기
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

### String 이어붙이기
아래 예제에서 `s1`은 더 이상 사용할 수 없다. `+` 연산자가 왼쪽 값인 `s1`의 ownership을 가져가기 때문이다. 그리고 `s3`를 만들 때 `s2` 앞에는 반드시 `&`를 붙여 `&String`으로 넘겨야 한다.
```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2;
```
사실 `+`는 `add()` 함수 호출로 변환된다.
```rust
s1.add(s2)
```
대략 이런 모양이다.
`add()` 함수의 signature를 보면:
```rust
fn add(self, s: &str) -> String {
```
인자로 `s: &str`을 받는다. 여기서는 `&String`이 deref coercion으로 `&str`로 변환되기 때문에 동작한다.


### Indexing

아래 코드는 에러가 난다. Rust의 string은 indexing을 지원하지 않기 때문이다.
```rust
let s1 = String::from("hello");
let h = s1[0];
```

### UTF-8
```rust
let hello = String::from("Hola");// 4 bytes
let hello = String::from("Здравствуйте");// 12 characters but 24 bytes
let hello = "Здравствуйте";
for n in hello.as_bytes(){
    println!("{}",n);

}
```

6개의 char가 출력된다.
```rust
for c in "नमस्ते".chars() {
    println!("{}", c);
}
```
18개의 byte가 출력된다.
```rust
for b in "नमस्ते".bytes() {
    println!("{}", b);
}
```
