+++
title="String in Rust"
date=2021-02-03

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### [Slice](https://doc.rust-lang.org/book/ch04-03-slices.html)  
스트링슬라이스 (string slice)는  String의 일부에 대한 참조자다

```rust
let s = String::from("hello world");

let hello = &s[0..5];//hello
let world = &s[6..11];//world
```

여기서 s는 불변 스트링 슬라이스
```rust
let s:&'static str = "Hello, world!";//data type is &'static str
let my_string:String = s.to_string()//data type is String
```

슬라이스 예제
```rust
let my_string = String::from("hello world");
&my_string[..]//string slice
let my_string_literal = "hello world";
my_string_literal//string slice
my_string_literal[..]//string slice
```

### [String](https://doc.rust-lang.org/book/ch08-02-strings.html)
rust의 core에서는 오직 하나의 string type을 지원하는데 그것이 바로 스트링슬라이스(str)다.
보통은 &str의 형태로 자주본다 

String타입은 Rust의 standard library가 제공하는 타입이다.
Rust의 standard library에는 String타입 외에도 OsString, OsStr, CString, CStr등을 제공한다. 

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

### String update하기
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
예제에서 s1은 더이상 사용불가하게 된다. 그리고 s3을 만들때 s2앞에 반드시 &를 붙여서 &String 타입으로 만들어 주어야 한다.
```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; 
```
사실 + 는 add()라는 함수로 치환되는데
```rust
s1.add(s2)
```
의 모양을 하고 있다  
잠깐 add()함수의 signature를 보고 가자
```rust
fn add(self, s: &str) -> String {
```
인자로 s:&str 타입을 받는데, &String을 &str로 강제 변환 해주기 때문에 신경쓰지 말고 넘어가자 


### Indexing

아래의 코드는 에러가 난다. rust의 스트링은 index를 지원하지 않기 때문이다
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

6개의 chars 이 프린트 된다
```rust
for c in "नमस्ते".chars() {
    println!("{}", c);
}
```
18개의 bytes 이 프린트 된다
```rust
for b in "नमस्ते".bytes() {
    println!("{}", b);
}
```
