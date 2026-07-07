+++
title="dyn and impl"
date=2023-05-22

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

`dyn`은 trait object를 표현하는 keyword다.
Rust 코드를 보다 보면 `dyn`이나 `impl`을 function parameter로 받거나 return type으로 쓰는 경우가 있다.
둘이 뭐가 다른지 찾아보다가 <https://cotigao.medium.com/dyn-impl-and-trait-objects-rust-fd7280521bea> 에 간단히 정리된 글이 있어 내용을 정리해 봤다.

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

`animal_talk` 함수는 인자로 `&dyn Animal`을 받는다.
여기서 `&`를 제거하면 compile time에 size를 알 수 없다고 compiler가 에러를 낸다.


만약 `&dyn` 대신 `impl`을 쓰면 어떨까?
실행 결과는 같다.
다른 점은 `impl`을 쓰면 compile time에 타입이 결정된다는 것이다. 내부적으로는 `Dog`를 받는 함수 하나, `Cat`을 받는 함수 하나처럼 구체 타입별 코드가 만들어진다.
그럼 generic type을 쓰는 함수와 `impl`을 쓰는 함수는 뭐가 다를까?
```rust
fn animal_talk<T: Animal>(a: T) {
  a.talk();
}
```
이것도 결과는 같다. parameter 위치에서는 크게 다르지 않아 보인다. 하지만 return type에서는 차이가 생긴다. 그냥 `Animal` trait 자체를 return할 수는 없다.
trait를 구현한 값을 return하려면 `impl Trait` 또는 `Box<dyn Trait>`를 사용해야 한다.

`impl Trait`를 return하는 함수를 만들어보자.
```rust
fn animal () -> impl Animal {
  if (is_dog_available()) {
    return Dog {};
  }
  Cat {}
}
```
이 코드는 실패한다. `impl Trait`는 compile time에 하나의 구체 타입으로 결정되어야 하기 때문이다. 즉 이 함수는 `Dog`나 `Cat` 중 하나의 타입만 return할 수 있다.


```rust
fn animal() -> Box<dyn Animal> {
  if (is_dog_available()) {
    return Box::new(Dog {});
  }

  Box::new(Cat {})
}
```
이렇게 하면 runtime에 trait object를 통해 dispatch되므로 여러 구체 타입을 return할 수 있다.
정리하면, 여러 타입 중 하나를 return하고 싶으면 `Box<dyn Trait>` 같은 trait object를 쓰고, 항상 하나의 구체 타입만 return한다면 `impl Trait`를 쓰면 된다.


<https://joshleeb.com/posts/rust-traits-and-trait-objects/> 도 같이 참고하면 좋다. 아직도 완전히 익숙하진 않다. 역시 C#이 편하다.
