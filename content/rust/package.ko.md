+++
title="Package manager"
date=2025-07-21

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

분명 며칠 전까지만 해도 잘 작동하던 `rust-analyzer`가 갑자기 안 돼서 어디에 설치되어 있는지 찾아봤다. 예전에도 `rust-analyzer` 문제로 고생했던 기억이 있어서, 다음번에 비슷한 일이 생겼을 때 보려고 오늘 사용한 명령어를 정리해 둔다.

먼저 `rust-analyzer`가 어디에 설치되어 있는지 확인한다.

```
which package_name
which rust-analyer
```
설치 위치는 `~/.cargo/bin/rust-analyzer`로 나온다.


## cargo
#### 설치된 프로그램 목록
```
cargo install --list
```
cargo로 설치된 프로그램 목록을 봤지만 `rust-analyzer`는 없었다.
그럼 cargo program들이 설치되는 위치로 가보자.
#### 프로그램 설치위치
```
ls ~/.cargo/bin/
```
`.cargo/bin`에 가보니 실제로 `rust-analyzer`가 있기는 한데, 파일 이름 마지막에 `@`가 붙어 있었다. 이건 symbolic link다. Windows의 바로가기와 비슷하다고 생각하면 된다.

#### 심볼릭 링크의 진짜 위치 찾기
```
readlink -f path
readlink ~/.cargo/bin/rust-analyzer
```
`rust-analyzer`의 실제 위치가 `rustup` 쪽이라는 것을 알 수 있다.


## rustup
`which rustup`으로 `rustup` 위치를 찾아보면 `~/.cargo/bin`에 있는 것을 알 수 있다. 실제 바이너리가 있는 위치라기보다는 symbolic link가 모여 있는 곳이다.
참고로 `rustup`의 metadata와 toolchain은 `~/.rustup`에 있다.

아마 예전에 `rust-analyzer`를 `rustup`으로 설치했다가 삭제하고 Homebrew로 다시 설치하면서 link가 `.cargo/bin`에 남아 있었던 것 같다.

그래서 다시 `rustup`으로 `rust-analyzer`를 설치하고, Homebrew로 설치했던 `rust-analyzer`는 삭제하기로 했다.

#### 프로그램 설치
```
rustup component add package_name
rustup component add rust analyzer
```

`component add`로 설치한 package는 어디에 설치될까?
보통 **`~/.rustup/툴체인/bin`** 아래에 있다.
내 경우에는 **`~/.rustup/toolchains/stable-aarch64-apple-darwin/lib`** 쪽에서 확인했다.



## Homebrew
먼저 Homebrew에 `rust-analyzer`가 설치되어 있는지 확인한다.
```
brew list
```
목록에 `rust-analyzer`가 있었다. 그런데 왜 `which rust-analyzer`를 했을 때는 `.cargo` 아래로 나오는지 모르겠다. 그동안은 왜 잘 작동했는지도 모르겠다. 최근에 했던 `brew upgrade` 때문이었을까?

어쨌든 `brew install`로 설치한 program의 실제 binary는 보통 아래 폴더 중 하나에 있다.
```
ls /opt/homebrew/bin
ls /opt/homebrew/opt
ls /opt/homebrew/Cellar
```
이제 Homebrew로 설치했던 `rust-analyzer`를 제거한다.
```bash
brew uninstall package_name
brew uninstall rust-analyzer
```
Helix로 Rust 파일을 열어보니 이제야 정상적으로 작동한다.
