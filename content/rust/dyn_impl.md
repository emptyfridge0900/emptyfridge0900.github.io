+++
title="dyn and impl"
date=2023-05-22

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

dyn는 trait object를 표현하는 keyword다.
가끔보면 dyn나 impl을 function parameter로 받거나 return하는 경우가 있다.
뭐가 다른거지 하고 검색하다가 https://cotigao.medium.com/dyn-impl-and-trait-objects-rust-fd7280521bea 에 간단하게 정리되있어서 가져와봤다.

```rust
trait Animal {
   fn talk(&self);
}
struct Cat {}
struct Dog {}
impl Animal for Cat {
  fn talk(&self) {
    println!(“meow”);
  }
}
impl Animal for Dog {
  fn talk(&self) {
    println!(“bark”);
  }
}
fn animal_talk(a: &dyn Animal) {
  a.talk();
}
fn main() {
  let d = Dog {};
  let c = Cat {};
  animal_talk(&d);
  animal_talk(&c);
}
```

animal_talk 함수를 보면 인자로 &dyn Animal 을 받는다.
&을 제거하면 compile time에 size를 알수 없다고 컴파일러가 징징거린다.


만일 &dyn 대신 impl를 쓰면 어떨까?
똑같은 결과가 나온다. 
대신 다른 점은 impl 쓰면 compile time에 타입이 결정되고 Dog를 받는 함수 하나, Cat을 받는 함수 하나를 만들어낸다.
그럼 generic type을 쓰는 함수와 impl을 쓰는 함수는 뭐가 다를까?
```rust 
fn animal_talk<T: Animal>(a: T) {
  a.talk();
}
```
똑같은 결과가 나온다. 다를게 없어보인다. 하지만 Animal 타입으로 리턴은 안된다.
그러면 generic 타입을 리턴하려면 어떻게 할까?
impl이나 Box< dyn Trait>를 사용해야한다.

impl Trait를 return하는 함수를 만들어보자
```rust
fn animal () -> impl Animal {
  if (is_dog_available()) {
    return Dog {};
  }
  Cat {}
}
```
fail한다. 왜냐면 impl은 타입을 comile time에 결정하기 때문에 Dog나 Cat 타입 둘중 하나만 리턴할수 있기 때문이다. 


```rust
fn animal() -> Box<dyn Animal> {
  if (is_dog_available()) {
    return Box::new(Dog {});
  } 
    
  Box::new(Cat {})
}
```
이렇게 하면 runtime에 타입이 결정되기 때문에 여러 타입을 리턴하고 싶으면 trait object을 사용하고 하나의 타입만 리턴한다면 impl을 쓰자.


https://joshleeb.com/posts/rust-traits-and-trait-objects/ 이것도 참조해서 보자. 아직도 완전히 이해가 가지 않는다. 역시 C#이 편하다.