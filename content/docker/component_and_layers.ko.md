+++
title = "Docker 구성요소와 레이어: docker, dockerd, containerd, runc, Colima, Lima"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "colima", "lima", "containerd"]
+++

Colima를 쓰는 맥에서 `docker ps`를 쳤을 때 실제로 무슨 일이 일어나는지, 그리고
docker / Docker Desktop / Colima / Lima / dockerd / containerd / runc 이 녀석들이
어떻게 레이어로 맞물려 돌아가는지 정리해봤다.

## 시리즈 목차

각 레이어를 더 자세히 다룬 글들. (5번째 레이어인 Linux 커널은 별도 글 없이 여기서만 다룬다.)

1. [Layer 1 — docker CLI (클라이언트)](./layer1_docker_cli.md)
2. [Layer 2 — dockerd (데몬 / API 서버)](./layer2_dockerd.md)
3. [Layer 3 — containerd (컨테이너 런타임)](./layer3_containerd.md)
4. [Layer 4 — runc (저수준 OCI 런타임)](./layer4_runc.md)
5. Layer 5 — Linux 커널 (이 글에서 다룸)

## 발단이 된 버그

`docker ps`를 쳤더니 ``Command `docker` not found``가 떴다. 원인은 Docker가
**아니었다**:

- Colima는 설치돼 있고 잘 돌아가고 있었다 (`runtime: docker`).
- `docker` CLI도 Homebrew로 설치돼 있었다
  (`/opt/homebrew/Cellar/docker/<ver>/bin/docker`).
- 그런데 `/opt/homebrew/bin/`으로 **심볼릭 링크가 안 걸려 있어서** `$PATH`에
  없었던 것이다.

해결:

```bash
brew link --overwrite docker
```

`brew link`는 formula가 설치된 파일들(이른바 "Cellar")을 Homebrew의 `bin/`
(즉 `$PATH`에 들어있는 곳)으로 심볼릭 링크를 걸어준다. `--overwrite`는 목적지에
이미 다른 파일이 있으면 그걸 덮어쓴다 — 예전에 깔았던 Docker Desktop의 잔재일
가능성이 높다 (실제로 `~/.docker` 디렉터리가 아직 남아 있었다).

## 핵심 개념

Docker/OCI 컨테이너는 **Linux** 기술이다 — Linux 커널이 필요하다. 맥에는 Linux 커널이
없으니까, 맥에서는 **CLI 아래의 모든 것이 "숨겨진 Linux VM을 만들고 맥이 거기에
말을 걸 수 있게" 하기 위해 존재한다.**

## 전체 스택 (위에서 아래로)

```
┌─────────────────────────────────────────────────────────────┐
│  1. docker CLI            "클라이언트" — 내가 입력하는 것       │  ← 나
│     (docker ps, run...)                                       │
│            │ socket을 통해 Docker API(HTTP)로 말한다            │
│            ▼                                                   │
├─────────────────────────────────────────────────────────────┤
│  2. dockerd               "서버/데몬" — API 요청을 받고          │
│     (Docker daemon)        이미지/네트워크를 관리                │
├─────────────────────────────────────────────────────────────┤
│  3. containerd            컨테이너 런타임 — 이미지 pull,         │
│                            컨테이너 라이프사이클 관리            │
├─────────────────────────────────────────────────────────────┤
│  4. runc                  저수준 OCI 런타임 — 커널 기능으로      │
│                            실제로 컨테이너를 생성                │
│                            (namespace, cgroup)                 │
├─────────────────────────────────────────────────────────────┤
│  5. Linux 커널            컨테이너가 진짜로 필요로 하는 것        │
│     (맥에서는 VM 안에 존재)                                     │
└─────────────────────────────────────────────────────────────┘
```

Docker Engine 자체는 "server + api + client" (레이어 1–2)다. 원래 Docker는
**모놀리식**이었지만 (레이어 2–4가 한 프로그램), Docker가 아키텍처를 모듈화하면서
`containerd`가 분리되어 나왔다 (Docker 1.11, 2016). 이후 Kubernetes의
CRI(Container Runtime Interface)가 이 분리를 더욱 강화했고, 덕분에 Kubernetes
같은 것들이 전체 Docker engine 없이도 `containerd`를 직접 쓸 수 있게 됐다.

### 각 레이어가 맡는 일

| # | 구성요소 | 하는 일 |
|---|---|---|
| 1 | **docker CLI** | *클라이언트*. 내 명령을 Docker API(HTTP) 요청으로 바꿔 보낸다. 자기 스스로는 아무것도 실행하지 않는다. |
| 2 | **dockerd** | *서버/데몬*. API 요청을 받아 이미지·네트워크·볼륨을 관리한다. 실제 실행은 아래로 위임한다. |
| 3 | **containerd** | *컨테이너 런타임*. 이미지를 pull/저장하고, 컨테이너 상태를 추적하며 라이프사이클을 감독한다. |
| 4 | **runc** | *저수준 런타임*. Linux 커널 기능(namespace + cgroup)으로 실제로 컨테이너를 만든다. 컨테이너 시작 때 잠깐 돌고 끝난다. |
| 5 | **Linux 커널** | 컨테이너의 본질: Linux 커널 위에서 격리된 프로세스. 맥에서는 VM 안에만 존재한다. |

## socket — CLI와 데몬을 잇는 통로

docker CLI와 데몬은 별개의 프로그램이다. 둘은 **Unix domain socket**으로 대화하는데,
이건 파일처럼 보이지만 데이터를 저장하지 않는다 — 살아있는 연결 지점이다
("서랍이 아니라 문").

```
$ ls -l ~/.colima/default/docker.sock
srw------- 1 bds0900 staff 0 Jun 11 13:05 ...docker.sock
│          └─ 크기 0: 저장된 게 없다. 데이터는 *통과*할 뿐
└─ 맨 앞 's' = socket (파일 '-' 도 디렉터리 'd' 도 아님)
```

- `s` 플래그 → socket 확정 (`stat`은 `type: Socket`, `file`은 `socket`이라 한다).
- 크기 `0` → 저장소가 아니라 통로다.
- `srw-------` → Colima 환경에서는 소유자만 데몬에 말을 걸 수 있다. 표준 Linux Docker는 `srw-rw----`로 `docker` 그룹 멤버도 접근을 허용한다.

`docker ps`를 치면 CLI는 이 socket을 열어 HTTP 요청(`GET /containers/json`)을 쓰고
응답을 읽는다. socket 파일은 **맥**에 있지만, 그게 닿는 데몬은 **Linux VM** 안에서
돈다 — 그 경계를 넘겨주는 일을 Colima가 한다.

## Docker Desktop vs Colima vs Lima

이 셋은 새로운 레이어가 아니다. **레이어 2–5(VM + 그 안의 데몬 스택)를 제공하는
세 가지 다른 방법**일 뿐이다. 맨 위의 `docker` CLI(레이어 1)는 셋 다 동일하다.

```
        ┌──────────── docker CLI (레이어 1) ────────────┐
        │   아래에서 돌고 있는 데몬에게 말을 건다          │
        └───────────────────────────────────────────────┘
                            ▲  ▲  ▲
          ┌─────────────────┘  │  └──────────────────┐
 ┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
 │ Docker Desktop │  │     Colima       │  │   (raw) Lima     │
 │ 자체 VM + 데몬 │  │  Lima를 감쌈 ────┼──┼──► Linux VM을    │
 │ 전체 + GUI     │  │  ("런타임 프록시")│  │   제공            │
 └────────────────┘  └──────────────────┘  └──────────────────┘
```

- **Lima** (*Li*nux *ma*chines)는 "Linux 가상머신 레이어를 제공"한다. 목표 자체가
  *containerd를 널리 쓰이게 하는 것*이다.
- **Colima** (*Co*ntainers on *Lima*)는 "컨테이너 런타임 실행 프록시"로 **Lima를
  감싼다** — Lima한테 "Linux VM 만들고, 데몬 깔고, socket 노출해줘"라고 시킨다.
- **Docker Desktop**은 레이어 2–5에 해당하는 것을 자체적으로 다 번들하고, 여기에
  GUI·설정 UI·자동 업데이트·Compose·옵션 Kubernetes 클러스터까지 얹는다.

| | **Docker Desktop** | **Colima** | **Lima (raw)** |
|---|---|---|---|
| VM 제공 | 자체 VM | **Lima** 경유 | **자기 자신** |
| dockerd/containerd 제공 | 번들 | VM 안에 세팅 | 직접 설정 |
| 지향점 | GUI + 올인원 | 컨테이너 우선, CLI | 범용 Linux VM |
| 관계 | 독립 | **Colima → Lima → VM** | 토대 |

Docker Desktop에서 Colima로 넘어가는 주된 이유는 **라이선스**(규모 큰 회사는
Docker Desktop 유료 구독이 필요하지만 Colima는 무료 OSS)와 **가볍고 CLI 중심**
(항상 떠 있는 GUI 앱이 없음)이라는 점이다.

## 각 슬롯을 채우는 제품들

모든 레이어는 고정된 제품이 아니라 *갈아끼울 수 있는 슬롯*이다:

| # | 레이어 (역할) | 표준 도구 | 다른 제품들 |
|---|---|---|---|
| 1 | 클라이언트 (CLI) | `docker` | `nerdctl`, `podman`, `crictl` |
| 2 | 데몬 / API 서버 | `dockerd` | Podman(데몬리스) |
| 3 | 런타임 (고수준) | `containerd` | CRI-O |
| 4 | 런타임 (저수준, OCI) | `runc` | `crun`, gVisor(`runsc`), Kata, `youki` |
| 5 | Linux 커널 | Linux | (VM 안에서 제공) |
| — | VM 엔진 | **Docker Desktop** | Lima, Rancher Desktop, Podman Machine, Minikube |
| — | 컨테이너 래퍼 | **Docker Desktop** | Colima, Rancher Desktop, OrbStack |
| — | 하이퍼바이저 (macOS) | Apple Virtualization.framework | Hypervisor.framework, QEMU |

## 이 맥에 매핑하면

```
 레이어 1  클라이언트         →  docker            (Homebrew, /opt/homebrew/bin/docker)
 ─ socket ─                 →  docker.sock       (~/.colima/default/docker.sock)
 레이어 2  데몬/API          →  dockerd           (VM 안)
 레이어 3  런타임 (고수준)    →  containerd        (VM 안)
 레이어 4  런타임 (저수준)    →  runc              (VM 안)
 레이어 5  Linux 커널        →  Linux (aarch64)   (VM 안)
 ─────────────────────────────────────────────────────────────────
 VM 래퍼                    →  Colima            ("런타임 프록시")
 VM 엔진                    →  Lima              (VM을 부팅)
 하이퍼바이저               →  Apple Virtualization.framework
```

`colima status`를 보면 VM이 `aarch64`에서 *macOS Virtualization.Framework*를
쓰고, docker socket은 `~/.colima/default/docker.sock`에 있다고 나온다.

VM 안에서 레이어 2–4가 도는 걸 보려면:

```bash
colima ssh -- sh -c 'ps -e -o pid,comm | grep -E "dockerd|containerd|runc"'
# dockerd와 containerd는 계속 떠 있다.
# runc는 보통 안 보인다 — 컨테이너 시작 시 잠깐 돌고 바로 끝나기 때문
# (docker run 도중에만 잡힌다).
```

## 한 문단 요약

컨테이너를 실제로 돌리는 **수직 스택은 하나**다:
`docker CLI → dockerd → containerd → runc → Linux 커널`. 맥에서는 그 커널이
**Linux VM** 안에만 존재한다. **Docker Desktop, Colima, Lima는 이 스택의 단계가
아니라, 이 스택을 *제공하는* 경쟁 방식들이다.** Lima는 VM을 제공하고, Colima는
Lima를 부려서 데몬까지 세팅하는 컨테이너 중심 래퍼이며, Docker Desktop은 자체
VM + 데몬을 번들한 올인원 GUI 앱이다. 맨 위의 `docker` CLI는 무엇을 골랐든
상관하지 않는다 — 그냥 socket을 열고, 듣고 있는 데몬에게 Docker API로 말할 뿐이다.

## 참고

- emptyfridge.dev — Docker 구성요소 아키텍처: <https://emptyfridge.dev/docker/docker/>
