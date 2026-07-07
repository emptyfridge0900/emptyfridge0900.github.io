+++
title="인터프리터"
date=2024-03-18

[taxonomies]
categories = ["Side Project"]
tags = ["project", "rust"]
+++


[Monkey programming language](https://github.com/emptyfridge0900/interpreter)

약 한 달 전에 이 [영상](https://youtu.be/yeatOU5vVsA?t=631)을 보고 interpreter를 공부해 보고 싶어졌다.

몇 달 전 체스 게임을 구현하면서 게임 프로그래밍은 지금 내 수준에서 도전하기에 난도가 높다는 것을 느꼈다. 그래서 이번에는 [책](https://interpreterbook.com/)을 따라가며 만들 수 있는 interpreter 구현부터 시작하기로 했다.

아마 4월까지는 퇴근 후 시간을 대부분 이 프로젝트에 쓰게 될 것 같다.

\+\+ update 2025-01-09 \+\+

프로젝트는 정말 재미있었고, interpreter라는 주제도 흥미로웠다. \
다음에는 이 프로젝트를 발판 삼아 compiler에 도전해 보고 싶다.

웹 서비스로 만들 때는 [Game of Life 프로젝트](https://emptyfridge.dev/projects/game-of-life)처럼 `wasm_bindgen`을 쓰려고 했지만, 최종적으로는 [Dioxus](https://dioxuslabs.com)라는 좋은 framework를 사용해 웹 서비스를 만들었다.

이 과정에서 가장 크게 배운 점은 trait object의 lifetime을 명시하지 않으면 기본적으로 `'static`으로 추론된다는 것이다. 이 사실을 몰라서 Rust compiler와 몇 시간 동안 씨름했다.

인터프리터 웹 서비스는 여기서 실행해 볼 수 있다. [👉 실행해 보기](https://emptyfridge.dev/interpreter)
