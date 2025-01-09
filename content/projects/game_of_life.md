+++
title="Game of life"
date=2024-06-08

[taxonomies]
categories = ["Side Project"]
tags = ["project", "rust"]
+++

I worked on an interpreter project about 3 months ago. I wanted to turn my project into a web service, so I googled it. All I have to do is compile the rust project as a wasm file and publish it. So I started the game-of-life project to practice converting rust code to wasm file.

https://rustwasm.github.io/docs/book/ 

This project of mine follows the tutorial above. Except for the project settings and webpack settings, all other codes were used exactly as they were in the tutorial.

In the tutuorial, author use wasm-pack-template to start the project but both game-of-life booka and wasm-pack-template are outdated, I couldn't learn much about setup and running the project. 
I depend a lot [wasm-pack doc](https://rustwasm.github.io/docs/wasm-pack/) while doing this project.


### Github Repository
[Click here](https://github.com/emptyfridge0900/game-of-life)

### Github Page Publishing
[Click here](https://emptyfridge0900.dev/game-of-life)
### References how to publish results on a GitHub page
#### Github Action
https://www.daleseo.com/?tag=GitHubActions

#### Github Action Workflow
https://github.com/TX-2/TX-2-simulator/blob/main/.github/workflows/deploy-wasm.yml
