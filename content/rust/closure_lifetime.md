+++
title="Closure lifetime"
date=2025-05-07

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

간만에 rust 공부를 하다가 예전에 공부했던 closure의 lifetime을 복습해야겠다는 생각이 들었다.

## 문제의 발단
몇달전 interpreter 프로젝트를 할때 제일 골치 아팠던게 evaluation을 끝낸 결과 같을 화면에 보여주는 일이었다.  
기존의 코드는 evaluation이 끝나면 마지막 결과 값만을 출력했는데 그렇게 마지막 값만 출력하면 string 을 출력하는 함수 put은 사실상 마지막 라인에서만 쓰일 수 있는 있으나 마나 한 함수 가 되어버리므로 put을 어디서든 쓸 수 있도록 업데이트 해야했다.  



## 해결책
그래서 찾아낸 방법은 evaluator안에 put 함수를 주입시켜서 evaluator가 소유한 메소드처럼 언제든지 중간중에 call을 할 수 있도록 만드는 것이였다. C#으로 치면 Func delegate를 evaluator의 constructor에 패스해 주어서 delegate를 필요할때마다 부르는 형식이다.
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
C#으로 말하자면 이런느낌?  

## 하지만...
모든 코드가 생각대로 구현되기만 하면 얼마나 좋을까? C#으로는 아주 쉽게 구현할 수 있는 해결책이였지만 rust로 구현하기에는 내 내공이 짧았다.
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
위의 코드는 내가 당시 테스트 하려고 썼던 rust 코드를 그대로 가져온 것이다.  
그리고 compiler가 뱉어내는 error 메세지는
> s1 does not live long enough  
> note: due to object lifetime defaults, Box<dyn for<'a> FnMut(&'a str)> actually means Box<(dyn for<'a> FnMut(&'a str) + 'static)> 

분명 s1의 lifetime은 프로그램이 끝날때까지이고 test.world()는 프로그램이 끝나기 전에 불리는데.  
하지만 note를 보면 closure가 static lifetime을 갖는다는 힌트를 볼 수 있다.  
왜인지 모르겠지만 기본 lifetime이 static이여서 그런건데 vector를 static으로 변환해보고 여기저기 lifetime을 변경해보았는데 잘 안됬다. 그래서 커뮤니티에 물어봤다.

https://users.rust-lang.org/t/lifetime-and-closure/123655


## 해결
jofas라는 행님이 10분만에 답을 달아주셨는데, 정답은 Box<dyn FnMut(&str)> lifetime을 Box<dyn FnMut(&str) + a`> 로 바꿔주는 것!  
Boxed closure의 lifetime이 Test 보다 오래 갈 수 없으니 문제 해결이다.
