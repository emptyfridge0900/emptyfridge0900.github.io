+++
title="Closure lifetime"
date=2025-05-07

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

While studying Rust again after a while, I decided to review the lifetime of closures that I had studied before.

## Where the problem started

A few months ago, while working on an interpreter project, one of the most annoying problems was showing evaluation results on the same screen after evaluation finished.

The existing code printed only the final result after evaluation ended. But if only the final value is printed, a string-output function like `put` can practically be used only on the last line, making it almost useless. So I needed to update it so `put` could be used anywhere.

## Solution idea

The method I found was to inject the `put` function into the evaluator so the evaluator could call it at any point, as if it owned the method. In C# terms, it is like passing a `Func` delegate into the evaluator's constructor and calling that delegate whenever needed.

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

In C#, it would feel something like this.

## But...

It would be nice if every idea could be implemented exactly as imagined. In C#, this solution would be very easy, but I did not have enough Rust skill to implement it cleanly.

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

The code above is the exact Rust test code I wrote at the time.

The compiler error was:

> s1 does not live long enough
> note: due to object lifetime defaults, Box<dyn for<'a> FnMut(&'a str)> actually means Box<(dyn for<'a> FnMut(&'a str) + 'static)>

I thought the lifetime of `s1` clearly lasted until the program ended, and `test.world()` was called before the program ended.

But the note hints that the closure has a static lifetime. For some reason, the default lifetime was static. I tried turning the vector into static and changing lifetimes in several places, but it did not work, so I asked the community.

<https://users.rust-lang.org/t/lifetime-and-closure/123655>

## Fix

Someone named jofas answered in about 10 minutes. The answer was to change the lifetime of `Box<dyn FnMut(&str)>` to `Box<dyn FnMut(&str) + 'a>`!

The boxed closure cannot outlive `Test`, so the problem is solved.
