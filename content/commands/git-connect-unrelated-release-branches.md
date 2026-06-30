+++
title = "Git unrelated histories: dev, stg, main 브랜치 연결하기"
date = 2026-05-23

[taxonomies]
categories = ["post"]
tags = ["git", "branch", "release", "github", "workflow"]
+++

회사 배포 브랜치 흐름은 보통 이런 식으로 흘러간다.

```text
feature branches -> dev -> stg -> main
```

여기서는 production 브랜치를 `main`이라고 부른다. 팀에 따라 이 브랜치 이름은 `prd`일 수도 있다.

내 경우 실제 환경에는 `int`도 있었지만, 핵심 문제는 `stg`와 `main` 사이였다. `dev`는 `stg`에서 파생된 브랜치라 둘은 Git history가 연결되어 있었다. 그런데 `main`은 따로 만들어진 브랜치라 `dev`, `stg`와 공통 조상이 없었다.

즉, 브랜치 이름으로는 같은 배포 흐름 안에 있는 것처럼 보였지만 Git 입장에서는 완전히 다른 프로젝트 두 개처럼 보이는 상태였다.

## 원하는 흐름

원하는 규칙은 단순했다.

```text
feature branches -> dev -> stg -> main
```

`main`은 feature branch를 직접 받으면 안 된다. `dev`에서 검증되고, `stg`를 통과한 코드만 `main`으로 올라가야 한다.

그래서 Git history도 이 방향을 이해할 수 있어야 한다.

```text
main <- stg <- dev
```

여기서 화살표는 "오른쪽 브랜치가 왼쪽 브랜치의 history를 포함한다"는 뜻이다.

- `stg`는 `main`의 history를 포함해야 한다.
- `dev`는 `stg`의 history를 포함해야 한다.
- 그래야 `dev -> stg`, `stg -> main` promotion PR이 정상적으로 동작한다.

## 문제: unrelated histories

`main`과 `stg`가 서로 관련 없는 history를 가지고 있으면 Git은 일반적인 merge로 둘을 연결하지 않는다.

이런 메시지를 만날 수 있다.

```bash
fatal: refusing to merge unrelated histories
```

이 상태에서 억지로 PR을 만들면 diff가 이상하게 보이거나, "이미 있는 파일이 전부 새 파일처럼 보이는" 상황이 생길 수 있다. Git이 두 브랜치 사이의 공통 기준점을 찾을 수 없기 때문이다.

## 해결 방향

목표는 파일 내용을 바꾸는 것이 아니었다. 이미 `stg`와 `dev`의 파일 내용은 그대로 유지해야 했다.

필요한 것은 딱 하나였다.

> 현재 브랜치의 파일 내용은 그대로 유지하면서, 다른 브랜치를 부모 history로만 연결한다.

이때 사용할 수 있는 merge 전략이 `-s ours`다.

`-s ours`는 conflict가 났을 때 일부 파일만 "ours"로 고르는 옵션이 아니다. merge commit은 만들지만, 결과 tree는 현재 브랜치의 파일 내용을 그대로 사용한다.

즉, 아래 작업은 `main`의 파일 내용을 `stg`에 덮어쓰는 것이 아니라, `main`을 `stg`의 history에 연결만 한다.

## 1. main을 stg에 연결하기

먼저 실제 `stg`에 바로 작업하지 말고 테스트 브랜치를 만든다.

```bash
git fetch origin
git checkout -b test-connect-main-stg origin/stg
```

그리고 `origin/main`을 merge하되, 현재 브랜치인 `stg`의 파일 내용을 그대로 유지한다.

```bash
git merge origin/main --allow-unrelated-histories -s ours
```

이제 반드시 diff를 확인한다.

```bash
git diff origin/stg..test-connect-main-stg
```

출력이 없어야 한다. 출력이 없다는 것은 merge commit은 생겼지만 파일 내용은 기존 `stg`와 같다는 뜻이다.

diff가 비어 있다면 실제 `stg`를 fast-forward로 이동한다.

```bash
git checkout stg
git merge --ff-only test-connect-main-stg
git push origin stg
```

이 작업이 끝나면 `origin/main`은 `origin/stg`의 조상 history가 된다.

```text
main <- stg
```

## 2. stg를 dev에 연결하기

다음은 같은 방식으로 `stg`를 `dev`에 연결한다.

```bash
git checkout -b test-connect-stg-dev origin/dev
```

현재 브랜치인 `dev`의 파일 내용은 유지하면서 `origin/stg`를 history로 연결한다.

```bash
git merge origin/stg --allow-unrelated-histories -s ours
```

마찬가지로 diff를 확인한다.

```bash
git diff origin/dev..test-connect-stg-dev
```

출력이 없어야 한다.

문제가 없다면 실제 `dev`를 fast-forward한다.

```bash
git checkout dev
git merge --ff-only test-connect-stg-dev
git push origin dev
```

이제 history 구조는 원하는 모양이 된다.

```text
main <- stg <- dev
```

## 이후 배포 흐름

이제부터는 promotion PR을 정상적으로 만들 수 있다.

```text
feature branch -> dev
dev -> stg
stg -> main
```

자동화한다면 이런 흐름이 된다.

```text
merge into dev
  -> auto PR: dev to stg
  -> after approval/merge
  -> auto PR: stg to main
```

실제 파이프라인에 `int`가 있다면 같은 원리로 중간에 넣으면 된다.

```text
main <- stg <- int <- dev
```

중요한 것은 production에 가까운 브랜치부터 개발 브랜치 방향으로 history를 연결하는 것이다. 그래야 promotion 방향인 `dev -> int -> stg -> main`의 PR diff가 자연스럽게 계산된다.

## 주의할 점

`-s ours`는 강력하지만 위험할 수 있다. 이 전략은 상대 브랜치의 파일 내용을 merge 결과에 반영하지 않는다.

그래서 아래 조건을 만족할 때만 사용해야 한다.

1. 목적이 파일 변경이 아니라 Git history 연결일 때
2. 현재 브랜치의 파일 내용을 그대로 유지해야 할 때
3. 상대 브랜치의 파일 내용이 현재 브랜치에 반영되지 않아도 되는 것이 명확할 때
4. 테스트 브랜치에서 diff가 비어 있음을 확인했을 때

특히 `main`에 실제 운영 코드가 있고, 그 내용이 `stg`에 없는 상태라면 이 방법을 그대로 쓰면 안 된다. 그런 경우에는 먼저 어떤 코드가 기준이 되어야 하는지 정해야 한다.

## 3줄 요약

브랜치 이름이나 배포 환경 이름은 사람이 이해하는 흐름일 뿐이다. Git이 이해하는 것은 commit graph다.

`dev`, `stg`, `main`이라는 이름을 붙였다고 해서 자동으로 promotion 관계가 생기지는 않는다. PR diff, merge 가능 여부, 자동 promotion은 결국 공통 조상 commit이 있는지에 영향을 받는다.

이번 문제의 핵심은 코드를 고치는 것이 아니라 history를 올바르게 연결하는 것이었다. 테스트 브랜치에서 `-s ours` merge를 만들고, diff가 비어 있는지 확인한 뒤, 실제 브랜치를 fast-forward하는 방식으로 안전하게 해결할 수 있었다.
