+++
title="Closure lifetime"
date=2025-05-07

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

오랜만에 Rust를 공부하다가 예전에 봤던 closure의 lifetime을 다시 복습해야겠다는 생각이 들었다.

## 문제의 발단
몇 달 전 interpreter 프로젝트를 할 때 가장 골치 아팠던 일은 evaluation 중간 결과를 화면에 보여주는 것이었다.
기존 코드는 evaluation이 끝난 뒤 마지막 결과값만 출력했다. 그렇게 되면 문자열을 출력하는 `put` 함수는 사실상 마지막 줄에서만 의미가 있고, 중간에서는 있으나 마나 한 함수가 된다. 그래서 `put`을 어디서든 쓸 수 있도록 바꿔야 했다.



## 해결책
그래서 찾은 방법은 evaluator 안에 `put` 함수를 주입해서, evaluator가 소유한 메서드처럼 중간중간 호출할 수 있게 만드는 것이었다. C#으로 치면 `Func` delegate를 evaluator의 constructor에 넘기고, 필요할 때마다 그 delegate를 호출하는 형태다.
```cs
public class Evaluator{
    private Func<string,string> _func;
    public Evaluator(Func<string,string> func){
        _func = func;
    }
    public void PrintToScreen(string msg){
        _func(msg);
    }
}
Func<string,string> print = (string msg)=> Console.WriteLine(msg);
var evaluator = new Evaluator(print);
evaluator.PrintToScreen("hello world!");
```
C#으로 말하면 대략 이런 느낌이다.

## 하지만...
모든 코드가 생각대로 구현되기만 하면 얼마나 좋을까? C#에서는 아주 쉽게 구현할 수 있는 해결책이었지만, Rust로 구현하기에는 내 내공이 부족했다.
```rs
fn main() {
    let mut s1:Vec<String> = vec!["hello".to_string()];
    let closure =  |str:&str| {
        s1.push(str.to_string());
    };
    let boxed = Box::new(closure);
    let mut test = Test::new(boxed);
    test.world("world");
    println!("{:?}",s1);
}

pub struct Test{
    output:Box<dyn FnMut(&str)>
}
impl Test{
    pub fn new(output:Box<dyn FnMut(&str)>)->Test{
        Test { output }
    }
    pub fn world(&mut self,str:&str){
        (self.output)(str);
    }
}
```
위 코드는 당시 테스트하려고 작성했던 Rust 코드를 그대로 가져온 것이다.
compiler가 뱉은 error message는 다음과 같았다.
> s1 does not live long enough
> note: due to object lifetime defaults, Box<dyn for<'a> FnMut(&'a str)> actually means Box<(dyn for<'a> FnMut(&'a str) + 'static)>

내 생각에는 `s1`의 lifetime이 `main`이 끝날 때까지이고, `test.world()`도 그 전에 호출되니 문제가 없어 보였다.
하지만 note를 보면 closure가 기본적으로 `'static` lifetime을 요구한다는 힌트가 있다.
처음에는 왜 기본 lifetime이 `'static`인지 잘 이해하지 못해서 vector를 static처럼 바꿔보거나 여기저기 lifetime을 붙여봤지만 잘 풀리지 않았다. 결국 커뮤니티에 질문했다.

https://users.rust-lang.org/t/lifetime-and-closure/123655


## 해결
jofas라는 분이 10분 만에 답을 달아주셨다. 정답은 `Box<dyn FnMut(&str)>`의 lifetime을 `Box<dyn FnMut(&str) + 'a>`로 바꾸는 것이었다.
boxed closure의 lifetime이 `Test`보다 오래 살아야 한다고 강제하지 않도록 만들면 문제가 해결된다.
