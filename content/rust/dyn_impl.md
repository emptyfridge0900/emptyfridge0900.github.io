+++
title="dyn and impl"
date=2023-05-22

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

`dyn` is the keyword used to represent a trait object.

Sometimes `dyn` or `impl` appears as a function parameter or return type.

I was wondering what the difference was and found a concise explanation at <https://cotigao.medium.com/dyn-impl-and-trait-objects-rust-fd7280521bea>, so I summarized it here.

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

The `animal_talk` function receives `&dyn Animal` as an argument.

If `&` is removed, the compiler complains that the size cannot be known at compile time.

What happens if we use `impl` instead of `&dyn`?

The result is the same. The difference is that with `impl`, the type is decided at compile time, and Rust generates one function for `Dog` and one function for `Cat`.

Then what is the difference between a function using a generic type and one using `impl`?

```rust
fn animal_talk<T: Animal>(a: T) {
  a.talk();
}
```

The result is the same. It does not seem different. But we cannot return the `Animal` type directly.

So how do we return a generic-looking type?

Use `impl` or `Box<dyn Trait>`.

Let's create a function that returns `impl Trait`.

```rust
fn animal () -> impl Animal {
  if (is_dog_available()) {
    return Dog {};
  }
  Cat {}
}
```

This fails because `impl` decides the type at compile time, so it can return only one of `Dog` or `Cat`.

```rust
fn animal() -> Box<dyn Animal> {
  if (is_dog_available()) {
    return Box::new(Dog {});
  }

  Box::new(Cat {})
}
```

This works because the type is decided at runtime. If you want to return multiple concrete types, use a trait object. If you return only one concrete type, use `impl`.

Also see <https://joshleeb.com/posts/rust-traits-and-trait-objects/>. I still do not fully understand it. C# is definitely more comfortable.
