+++
title="Game of Life"
date=2024-06-08

[taxonomies]
categories = ["Side Project"]
tags = ["project", "rust"]
+++

약 3개월 전에 interpreter 프로젝트를 진행했다. 그 프로젝트를 웹 서비스로 만들어 보고 싶어서 방법을 찾아보니, Rust 프로젝트를 WASM 파일로 컴파일해서 배포하면 된다는 것을 알게 되었다. 그래서 Rust 코드를 WASM으로 변환하는 과정을 연습하기 위해 Game of Life 프로젝트를 시작했다.

<https://rustwasm.github.io/docs/book>

이 프로젝트는 위 튜토리얼을 따라 만들었다. 프로젝트 설정과 webpack 설정을 제외하면, 나머지 코드는 튜토리얼의 코드를 거의 그대로 사용했다.

튜토리얼에서는 `wasm-pack-template`으로 프로젝트를 시작하지만, Game of Life 책과 `wasm-pack-template` 모두 오래되어 프로젝트 설정과 실행 방법을 제대로 배우기 어려웠다.
그래서 이 프로젝트를 진행할 때는 [wasm-pack 문서](https://rustwasm.github.io/docs/wasm-pack/)를 많이 참고했다.


[👉 GitHub Repository](https://github.com/emptyfridge0900/game-of-life)

[👉 Game of Life 실행해 보기](https://emptyfridge.dev/game-of-life)

### GitHub Pages 배포 참고 자료
#### GitHub Action
<https://www.daleseo.com/?tag=GitHubActions>

#### GitHub Action Workflow
<https://github.com/TX-2/TX-2-simulator/blob/main/.github/workflows/deploy-wasm.yml>
