+++
title="체스"
date=2024-03-04

[taxonomies]
categories = ["Side Project"]
tags = ["project", "rust"]
+++

# GitHub Repository
[여기서 보기](https://github.com/emptyfridge0900/chess)


### 목표
[The Rust Programming Language](https://doc.rust-lang.org/book/)에서 배운 내용을 실제 사이드 프로젝트에 적용해 보면서, 내가 Rust를 얼마나 이해하고 있는지 확인하는 것이 목표였다.


### 회고
이 프로젝트는 내가 처음으로 만든 Rust 프로젝트다. 체스 프로그램 구현은 처음 생각했던 것보다 훨씬 어려웠다. 특히 앙파상(En passant), 캐슬링(Castling), 프로모션(Promotion) 같은 특수 규칙을 구현하는 데 꽤 애를 먹었다.

반면 이 프로젝트를 하면서 Rust의 ownership에 대한 이해는 훨씬 단단해졌다. C# 배경만 있던 나에게 ownership은 이해한 것 같지만 막상 코드로 쓰면 완전히 이해하지 못했다는 사실이 드러나는 개념이었다.

`RefCell`과 `Box`를 사용하면서 smart pointer에도 조금 더 익숙해졌다. 그래도 smart pointer는 아직 더 공부해야 한다. 여전히 잘 이해하지 못한 개념은 lifetime과 concurrency다. 다음 Rust 프로젝트에서는 이 부분을 더 제대로 다뤄 보고 싶다.

전체적으로는 재미있는 프로젝트였다. 다만 처음에 체스 규칙 자체를 배워야 했던 과정은 생각보다 번거롭고 시간이 많이 들었다. 어떤 프로젝트든 마찬가지다. 비즈니스 로직을 충분히 이해하지 않은 채 구현부터 시작하면, 중간에 큰 수정이 필요해진다.

