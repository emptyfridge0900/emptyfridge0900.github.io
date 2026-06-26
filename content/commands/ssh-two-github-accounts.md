+++
title = "SSH로 GitHub 계정 두 개 동시에 쓰기 (회사 + 개인)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["git", "ssh", "github", "command"]
+++

GitHub 계정이 두 개(회사용, 개인용)인데 같은 맥에서 쓰려면 HTTPS 방식으로는 한계가 있다.
macOS Keychain이 `github.com` 당 토큰을 하나만 저장하기 때문에, 나중에 로그인한 계정의 토큰으로 항상 덮어씌워진다.

```
remote: Permission to emptyfridge0900/repo.git denied to bds0900.
fatal: unable to access '...': The requested URL returned error: 403
```

SSH 키를 계정별로 따로 만들고 `~/.ssh/config`로 라우팅하면 이 문제를 깔끔하게 해결할 수 있다.

---

## 전체 흐름

```
1. 기존 SSH 키 확인
2. 파일 이름 규칙 이해
3. 회사 계정용 SSH 키 새로 생성 (ssh-keygen)
4. ~/.ssh/config 에 Host 별칭 등록
5. SSH 에이전트에 키 추가 (ssh-add)
6. 회사 GitHub 에 공개키 등록
7. 연결 테스트 (ssh -T)
8. 기존 repo의 remote URL 을 SSH 방식으로 변경
9. 재부팅 후에도 유지되게 만들기 (AddKeysToAgent)
10. GitHub CLI(`gh`) 계정도 분리하기 (`GH_CONFIG_DIR`)
```

---

## 1. 기존 SSH 키 확인

```bash
ls -la ~/.ssh/
```

`id_ed25519` 와 `id_ed25519.pub` 가 있으면 이미 SSH 키가 하나 있는 것이다.
어느 계정에 연결되어 있는지는 공개키 파일 마지막 줄을 보면 알 수 있다.

```bash
cat ~/.ssh/id_ed25519.pub
# ssh-ed25519 AAAA... personal@gmail.com
```

GitHub 에 실제로 연결되는지도 확인한다.

```bash
ssh -T git@github.com
# Hi emptyfridge0900! You've successfully authenticated...
```

`ssh -T`는 **터미널(shell) 세션 없이 인증만 테스트**하는 옵션이다 (`-T` = no pseudo-terminal).

---

## 2. 파일 이름 규칙 이해

SSH 키 파일 이름은 두 부분으로 구성된다.

| 부분 | 의미 |
|---|---|
| `id_` | "identity"의 약자. SSH가 인증에 쓰는 파일임을 나타내는 표준 접두사 |
| `ed25519` | 암호화 알고리즘 이름 (Edwards-curve Digital Signature Algorithm 25519). 현재 가장 권장되는 방식 |

키를 생성하면 항상 두 파일이 만들어진다.

- `id_ed25519` — **개인키(private key)**. 절대 외부에 공유하지 않는다.
- `id_ed25519.pub` — **공개키(public key)**. GitHub 에 등록하는 파일.

---

## 3. 회사 계정용 SSH 키 생성

```bash
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_ed25519_company -N ""
```

각 옵션 설명:

| 옵션 | 의미 |
|---|---|
| `-t ed25519` | 알고리즘 타입 지정. `ed25519`가 현재 표준 (기존의 `rsa` 보다 짧고 빠르고 안전) |
| `-C "work@company.com"` | Comment. 공개키 파일 맨 끝에 붙는 **라벨**. 어떤 계정 키인지 구분하는 용도. 이메일일 필요는 없지만 관례적으로 사용 |
| `-f ~/.ssh/id_ed25519_company` | 저장할 파일 경로 지정. 기본값은 `~/.ssh/id_ed25519` 인데, 이미 개인용 키가 거기 있으니 이름을 다르게 준다 |
| `-N ""` | 패스프레이즈(암호)를 빈 값으로 설정. 입력창이 뜨지 않고 바로 생성된다 |

실행 후 두 파일이 생긴다.

```
~/.ssh/id_ed25519_company      # 개인키
~/.ssh/id_ed25519_company.pub  # 공개키
```

---

## 4. `~/.ssh/config` 설정

SSH 클라이언트는 연결 시 `~/.ssh/config` 를 읽어서 어떤 키를 쓸지 결정한다.
계정별로 `Host` 별칭을 만들어주면 된다.

```
# 개인 GitHub (emptyfridge0900)
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes

# 회사 GitHub (bds0900)
Host github-company
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_company
  AddKeysToAgent yes
```

각 지시어 설명:

| 지시어 | 의미 |
|---|---|
| `Host github-personal` | 이 블록에 붙이는 **별칭**. `git@github-personal:...` 처럼 쓰면 이 설정이 적용됨 |
| `HostName github.com` | 실제 접속할 서버 주소. 별칭이 달라도 실제로는 `github.com` 에 연결 |
| `User git` | SSH 로그인할 유저명. GitHub 은 항상 `git` 유저를 사용 (계정 구분은 키로 함) |
| `IdentityFile` | 이 별칭으로 접속할 때 사용할 개인키 경로 |
| `AddKeysToAgent yes` | 처음 사용 시 자동으로 SSH 에이전트에 키를 등록. 재부팅 후에도 `ssh-add` 를 수동으로 안 해도 됨 |

---

## 5. SSH 에이전트에 키 추가

```bash
ssh-add ~/.ssh/id_ed25519
ssh-add ~/.ssh/id_ed25519_company
```

**SSH 에이전트(`ssh-agent`)** 란 개인키를 메모리에 올려두고 필요할 때 꺼내주는 백그라운드 프로세스다.
키를 에이전트에 등록하면 매번 키 파일 경로를 지정하지 않아도 된다.

`ssh-add` 주요 옵션:

| 옵션 | 의미 |
|---|---|
| `~/.ssh/id_ed25519` | 등록할 개인키 파일 경로 |
| `-l` | 현재 에이전트에 등록된 키 목록 확인 (list) |
| `-d ~/.ssh/id_ed25519` | 에이전트에서 특정 키 제거 (delete) |
| `-D` | 에이전트의 모든 키 제거 |

등록 확인:

```bash
ssh-add -l
# 256 SHA256:xxx personal@gmail.com (ED25519)
# 256 SHA256:yyy work@company.com (ED25519)
```

> `ssh-add` 로 추가한 키는 재부팅하면 사라진다. `AddKeysToAgent yes` 를 config 에 넣으면 첫 사용 시 자동 재등록된다.

---

## 6. 회사 GitHub 에 공개키 등록

공개키를 클립보드에 복사한다.

```bash
cat ~/.ssh/id_ed25519_company.pub | pbcopy
```

GitHub → **Settings** → **SSH and GPG keys** → **New SSH key** 에서 붙여넣기 후 저장.

---

## 7. 연결 테스트

```bash
ssh -T git@github-personal
# Hi emptyfridge0900! You've successfully authenticated...

ssh -T git@github-company
# Hi bds0900! You've successfully authenticated...
```

`github-personal` / `github-company` 는 `~/.ssh/config` 에서 만든 별칭이다.
실제로는 둘 다 `github.com` 에 접속하지만, 서로 다른 키를 쓴다.

---

## 8. Remote URL 변경

### 기존 repo (HTTPS → SSH)

```bash
git remote set-url origin git@github-personal:emptyfridge0900/repo-name.git
```

`git remote -v` 로 확인:

```
origin  git@github-personal:emptyfridge0900/repo-name.git (fetch)
origin  git@github-personal:emptyfridge0900/repo-name.git (push)
```

### 회사 repo

```bash
git remote set-url origin git@github-company:bds0900/repo-name.git
```

### 새 repo 클론 시

```bash
# 개인 계정 repo
git clone git@github-personal:emptyfridge0900/repo-name.git

# 회사 계정 repo
git clone git@github-company:bds0900/repo-name.git
```

URL 구조: `git@<Host별칭>:<GitHub유저명>/<저장소이름>.git`

---

## 9. 재부팅 후에도 유지되게 만들기

`ssh-add` 로 에이전트에 등록한 키는 **재부팅하면 사라진다**. 터미널 세션이 닫히면 메모리에서 내려가기 때문이다.

`~/.ssh/config` 에 `AddKeysToAgent yes` 를 넣으면 해결된다.

```
Host github-personal
  ...
  AddKeysToAgent yes   # ← 이 줄

Host github-company
  ...
  AddKeysToAgent yes   # ← 이 줄
```

`AddKeysToAgent yes` 가 하는 일:

- 재부팅 후 처음으로 `git push` 나 `git pull` 을 실행하면 SSH가 `~/.ssh/config` 를 읽는다.
- 해당 Host 에 `AddKeysToAgent yes` 가 있으면, SSH가 **자동으로 `ssh-add` 를 대신 실행**해서 에이전트에 키를 올린다.
- 그 이후부터는 같은 세션에서 다시 물어보지 않는다.

즉, 재부팅 후 수동으로 `ssh-add` 를 칠 필요가 없다. 첫 git 명령 한 번이면 자동으로 로드된다.

> macOS 에서 `~/.ssh/config` 파일 자체는 영구적이다. 재부팅해도 설정은 그대로 남는다.
> 휘발되는 건 **에이전트 메모리(ssh-add 결과)** 뿐이고, `AddKeysToAgent yes` 가 그걸 자동으로 채워준다.

---

## 최종 구성 요약

| 계정 | Host 별칭 | 키 파일 |
|---|---|---|
| `emptyfridge0900` (개인) | `git@github-personal` | `~/.ssh/id_ed25519` |
| `bds0900` (회사) | `git@github-company` | `~/.ssh/id_ed25519_company` |

이제 `git push` / `git pull` 할 때 GitHub 계정을 의식하지 않아도 된다.
repo 의 remote URL 에 붙인 Host 별칭이 알아서 맞는 키를 골라 쓴다.

---
## 주가기능

## 10. GitHub CLI(`gh`) 계정도 분리하기

여기까지 설정하면 `git push` / `git pull` 은 계정별 SSH 키로 잘 분리된다.
하지만 `gh issue create`, `gh pr create` 같은 GitHub CLI 명령은 SSH 키가 아니라 GitHub API 토큰을 쓴다.
즉, remote URL 이 `git@github-personal:...` 이어도 `gh` 가 회사 계정 토큰으로 로그인되어 있으면 개인 repo 이슈 생성이 실패할 수 있다.

처음 떠오르는 해결책은 `gh auth switch` 이다.

```bash
gh auth switch -u emptyfridge0900
gh issue create -R emptyfridge0900/repo-name

gh auth switch -u bds0900
gh issue create -R bds0900/repo-name
```

하지만 매번 계정을 바꾸는 방식은 귀찮고 실수하기 쉽다.
더 나은 방법은 `gh` 설정 디렉터리를 계정별로 나누는 것이다.
`gh` 는 `GH_CONFIG_DIR` 환경변수로 설정 저장 위치를 바꿀 수 있다.

쉘 설정 파일(`~/.zshrc` 등)에 alias 를 추가한다.

```bash
alias ghp='GH_CONFIG_DIR=$HOME/.config/gh-personal gh'
alias ghc='GH_CONFIG_DIR=$HOME/.config/gh-company gh'
```

그 다음 각 alias 로 한 번씩만 로그인한다.

```bash
GH_CONFIG_DIR=$HOME/.config/gh-personal gh auth login -h github.com --git-protocol ssh
GH_CONFIG_DIR=$HOME/.config/gh-company gh auth login -h github.com --git-protocol ssh
```

이후부터는 계정을 전환하지 않고 명령어만 구분해서 쓰면 된다.

```bash
# 개인 계정으로 개인 repo 이슈 생성
ghp issue create -R emptyfridge0900/repo-name

# 회사 계정으로 회사 repo 이슈 생성
ghc issue create -R bds0900/repo-name
```

정리하면 역할이 이렇게 나뉜다.

| 도구 | 계정 분리 방법 | 예시 |
|---|---|---|
| `git` | SSH Host 별칭 | `git@github-personal:emptyfridge0900/repo-name.git` |
| `gh` | `GH_CONFIG_DIR` 별도 설정 | `ghp issue create -R emptyfridge0900/repo-name` |

이렇게 하면 `git` 도 `gh` 도 계정 전환 없이 사용할 수 있다.
개인 작업은 `github-personal` + `ghp`, 회사 작업은 `github-company` + `ghc` 로 고정하면 된다.
