+++
title="Package manager"
date=2025-07-21

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

분명 몇일전까지만 해도 잘 작동하던 rust-analyzer가 갑자기 안되서 어디에 설치되었나 찾아보았다. 사실 저번에도 rust-analyzer 문제가 있어서 고생했던 기억이 있는데, 다음번에도 비슷한 일이 일어날거 같아서 오늘 사용했던 명령어를 정리해두었다.

먼저 rust-analyzer가 어디에 설치되었는지 보자

```
which package_name
which rust-analyer
```
설치 위치는 ~/.cargo/bin/rust-analyzer 로 뜬다.


## cargo
#### 설치된 프로그램 리스트
```
cargo install --list
```
cargo install --list로 설치된 프로그램들을 보았지만 rust-analyzer는 없었다.
그럼 cargo 프로그램들이 설치되는 위치로 가보자.  
#### 프로그램 설치위치
```
ls ~/.cargo/bin/
```
.cargo/bin에 가보니 진짜 rust-analyzer가 있기는 한데 파일이름 마지막에 @가 붙어있다. 이건 심볼링크이다. 한마디로 윈도우에서의 바로가기랑 같은 것이라고 생각하면 된다.

#### 심볼릭 링크의 진짜 위치 찾기
```
readlink -f path
readlink ~/.cargo/bin/rust-analyzer
```
rust-analyzer의 실제 위치는 rustup인 것을 알 수 있다.


## rustup
rustup 기본위치를 **which rustup**로 찾아보면 ~/.cargo/bin인걸 알수 있다. 왜그런지는 모르겠지만 실제 위지는 아니고 심볼릭 링크만 모여져 있는 곳이다.  
참고로 rustup의 메타데이터와 툴체인은 ~/.rustup 에 있다.

아마 예전에 rust-analyzer를 rustup으로 설치했다가 삭제하고 homebrew로 다시 설치해서 링크가 .cargo/bin에 남아있었나보다.

그래서 다시 rustup으로 rust-analyzer 설치하고 homebrew로 설치했던 rust-analyzer 삭제하기로 했다.

#### 프로그램 설치
```
rustup component add package_name
rustup component add rust analyzer
```

component add로 설치한 패키지는 어디에 설치될까?
**~/.rustup/툴체인/bin**에 있다.
나같은 경우는 
**~/.rustup/toolchains/stable-aarch64-apple-darwin/lib** 에 있었다.



## Homebrew
먼저 homebrew에 rust-analyzer가 있는지 확인해보자
```
brew list
```
리스트에 rust-analyzer가 있다. 근데 왜 which rust-analyzer 했을때는 .cargo에 있다고 뜨는지 모르겠다...그리고 왜 그동안은 rust-analyzer가 잘 작동했는지도 모르겠다. 최근에 했던 brew ugrade 때문에 그런거려나  
쨋든 **brew install**로 설치했던 프로그램의 실제 바이너리는 아래 폴더중 하나에 있을 것이다.
```
ls /opt/homebrew/bin
ls /opt/homebrew/opt
ls /opt/homebrew/Cellar
```
이제 homebrew로 설치했던 rust-analyzer를 제거해주자
```bash
brew uninstall package_name
brew uninstall rust-analyzer
```
helix로 러스트 파일을 열어보니 이제서야 잘 작동한다.

