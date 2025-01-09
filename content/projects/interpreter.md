+++
title="Interpreter"
date=2024-03-18

[taxonomies]
categories = ["Side Project"]
tags = ["project", "rust"]
+++


[Monkey programming lanuage](https://github.com/emptyfridge0900/interpreter)  

About a month ago watched this [video](https://youtu.be/yeatOU5vVsA?t=631), and I wanted to learn about interpreter.

While implementing a chess game a few months ago, I realized that the level of game programming was too high for me to challenge myself, so I decided to start with building an interpreter that I could follow by reading a [book](https://interpreterbook.com/).

I'll probably spend my whole after work hours work on this project until April.

\+\+ update 2025-01-09 \+\+

Project was super fun and interpreter was an interesting topic! \
I will use this project as a stepping stone to challenge myself with compilers next time.

When making it into a web service, I tried to use wasm_bindgen like I did in the [game of life project](https://emptyfridge.dev/projects/game-of-life), but I created a web service using a great framework called [Dioxus](https://dioxuslabs.com).
The biggest lesson I learn from this is the lifetime of a Trait object is inferred as 'static when the lifetime is not there. I didn't know this and struggled with Rust's compiler for hours.

And here is my interpreter web service [👉Try it out](https://emptyfridge.dev/interpreter)