+++
title="Ownership"
date=2025-05-24

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### [Ownership Rules](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ownership-rules)
1. Each value in Rust has an owner.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value will be dropped.


There are a few rules for your code to comply ownership rule. And as a C# developer, these 3 were the most confusing.

#### Copy
```rs
let x = 5;
let y = x;
```
#### Move
```rs
let s1 = String::from("hello");
let s2 = s1;
```
#### Borrow
```rs
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
println!("{r1} and {r2}");
// Variables r1 and r2 will not be used after this point.

let r3 = &mut s; // no problem
println!("{r3}");
```

### Copy
When I read [the book](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) and its examples, I feel like I understood it. But when I started coding, I got confused between copy, borrow, and move. Recently, while coding in C#, I realized why I was confusing copy, borrow, and move.

The example below is similar to a problem I encountered a few days ago while coding. 

```cs
public class Apple{
    public string Name{get;set;}
    public int Total{get;set;}
    public Apple(string name, int qty){
        Name=name;
        Total=qty;
    }
}
public class Store{
    public Queue<Apple> AppleBasckets {get;set;}
    public Store(){
        AppleBasckets = new Queue<Apple>();
    }
    public Apple Take(int quantity){
        Apple result = null;
        if(AppleBasckets.TryPeek(out var apple)){
            if(apple.Name == "Fuji")
            {
                result = apple;
                apple.Total -= quantity;
            }
        }
        return result;
    }
}
public static void Main(string[] args)
{
    var store = new Store();
    store.AppleBasckets.Enqueue(new Apple("Fuji",40));
    store.AppleBasckets.Enqueue(new Apple("Ambrosia",20));
    
    var result = store.Take(5);
    Console.WriteLine(result.Total);
}
```
This prints 35 in the console. I had unconsciously assumed that assignment operator (=) meant a copy operation. My intention was to copy the value of __apple__ to __result__ when doing **result = apple;**, but this caused unintended modification because it only copies the reference, and not the object's data. In Rust terminology, this is called a mutable borrow. If you want a copy of an object in C#, you need to implement [ICloneable](https://learn.microsoft.com/en-us/dotnet/api/system.icloneable?view=net-9.0).



### Move
In the C# code above, one piece of data was referenced by two variables. This corresponds to Figure 1-1. In Rust, data is handled as shown in Figure 1-3, following the ownership model.



<div style="display: flex; justify-content: space-around; align-items: center;">
  <div style="text-align: center;">
    <img src="https://doc.rust-lang.org/book/img/trpl04-02.svg" alt="Description" width="200" height="200">
    <p>[Figure 1-1]: 1 data 2 owner</p>
  </div>
  <div style="text-align: center;">
    <img src="https://doc.rust-lang.org/book/img/trpl04-03.svg" alt="Description" width="200" height="200">
    <p>[Figure 1-2]: 2 data 2 owner</p>
  </div>
  <div style="text-align: center;">
    <img src="https://doc.rust-lang.org/book/img/trpl04-04.svg" alt="Description" width="200" height="200">
    <p>[Figure 1-3]: 1 data 1 owner</p>
  </div>
</div>

*All the images above are from [Rust programming language book](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ownership-rules)

### Lesson I learned
Whenever I encounter the assignment operator (=), I should consider whether it performs a copy or just copy a reference, whether in C#, JavaScript, or Rust.