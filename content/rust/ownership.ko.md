+++
title="Ownership"
date=2025-05-24

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

### [Ownership Rules](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ownership-rules)
1. Rust의 모든 값에는 owner가 있다.
2. 한 번에 owner는 하나만 존재할 수 있다.
3. owner가 scope 밖으로 나가면 값은 drop된다.


ownership rule을 따르려면 몇 가지 개념을 구분해야 한다. C# 개발자 입장에서 가장 헷갈렸던 것은 아래 세 가지였다.

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
[The Book](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)의 예제를 읽을 때는 이해한 것 같았다. 하지만 실제로 코딩을 시작하니 copy, borrow, move가 계속 헷갈렸다. 최근 C# 코드를 작성하다가 내가 왜 이 셋을 헷갈렸는지 깨달았다.

아래 예제는 며칠 전 실제로 코딩하다가 만난 문제와 비슷하다.

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
콘솔에는 `35`가 출력된다. 나는 무의식적으로 assignment operator(`=`)가 copy를 의미한다고 생각하고 있었다. `result = apple;`을 할 때 `apple`의 값을 `result`로 복사하고 싶었지만, 실제로는 object data가 아니라 reference만 복사됐다. 그래서 의도치 않게 원본 객체가 수정됐다.

Rust 식으로 생각하면 이 상황은 "같은 데이터를 여러 변수가 가리키고 있고, 그중 하나를 통해 값을 변경했다"에 가깝다. C#에서 객체 자체를 복사하고 싶다면 [ICloneable](https://learn.microsoft.com/en-us/dotnet/api/system.icloneable?view=net-9.0)을 구현하거나 별도의 copy 로직을 작성해야 한다.



### Move
위 C# 코드에서는 하나의 데이터를 두 변수가 참조했다. 아래 그림으로 보면 Figure 1-1에 해당한다. Rust는 ownership model을 따르기 때문에 Figure 1-3처럼 데이터를 다룬다.



<div style="display: flex; justify-content: space-around; align-items: center;">
  <div style="text-align: center;">
    <img src="https://doc.rust-lang.org/book/img/trpl04-02.svg" alt="Description" width="200" height="200">
    <p>[Figure 1-1]: 데이터 1개, owner 2개처럼 보이는 상태</p>
  </div>
  <div style="text-align: center;">
    <img src="https://doc.rust-lang.org/book/img/trpl04-03.svg" alt="Description" width="200" height="200">
    <p>[Figure 1-2]: 데이터 2개, owner 2개</p>
  </div>
  <div style="text-align: center;">
    <img src="https://doc.rust-lang.org/book/img/trpl04-04.svg" alt="Description" width="200" height="200">
    <p>[Figure 1-3]: 데이터 1개, owner 1개</p>
  </div>
</div>

*위 이미지는 모두 [The Rust Programming Language](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ownership-rules)에서 가져왔다.

### 배운 점
assignment operator(`=`)를 볼 때마다 이 연산이 실제 데이터를 복사하는지, reference만 복사하는지, 아니면 ownership을 move하는지 먼저 생각해야 한다. C#, JavaScript, Rust 모두 이 차이를 의식해야 한다.
