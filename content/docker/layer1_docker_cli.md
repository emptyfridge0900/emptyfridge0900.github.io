+++
title = "Layer 1 — docker CLI (클라이언트)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "cli", "layers"]
+++

> Docker 레이어 시리즈 1편. 스택은 위에서 아래로:
> **1. docker CLI (클라이언트)** → 2. dockerd (데몬) → 3. containerd → 4. runc →
> 5. Linux 커널. [개요 글](./component_and_layers.md) 참고.

## 정체

`docker` CLI는 내가 명령을 입력하는 프로그램이다 — `docker ps`, `docker run`,
`docker build`. 얘는 **클라이언트일 뿐이다**. 직접 컨테이너를 돌리거나 빌드하거나
저장하지 않는다. 하는 일은 오직 내 명령을 HTTP 요청으로 바꿔서 데몬에게 보내고,
응답을 받아 화면에 보기 좋게 뿌리는 것뿐이다.

쓸 만한 비유: CLI는 **리모컨**이다. 버튼을 누르면 신호를 보내고, 실제 일은 TV가
한다. `docker`가 리모컨, `dockerd`가 TV다.

## 어디에 있나

```bash
$ which docker
/opt/homebrew/bin/docker          # Homebrew 심볼릭 링크...
$ ls -l /opt/homebrew/bin/docker
... -> ../Cellar/docker/29.5.3/bin/docker   # ...실제 바이너리로 연결
```

이 맥에서는 CLI가 Docker Desktop이 아니라 **Homebrew**(`brew install docker`)에서
왔다. 처음의 "command not found" 버그가 바로 이 구분 때문이었다: Docker Desktop은
예전엔 자체 `docker` CLI를 번들했는데, 그걸 지우면서 번들 CLI도 같이 사라졌다.
Homebrew로 깐 독립 CLI는 존재했지만 `brew link --overwrite docker`를 하기 전까지
`$PATH`에 링크가 안 걸려 있었다.

**핵심:** CLI는 엔진과 독립적이다. 같은 `docker` 바이너리가 Docker Desktop,
Colima, 원격 서버 — Docker API만 말할 수 있으면 무엇에든 붙는다.

## 명령을 치면 일어나는 일

`docker ps`는 로컬에서 "컨테이너 목록" 코드가 도는 게 아니다. API 호출이다:

```
docker ps
  └─ CLI가 HTTP 요청을 만든다:    GET /v1.43/containers/json
  └─ socket을 연다:              unix:///Users/you/.colima/default/docker.sock
  └─ 요청을 쓰고, JSON 응답을 읽는다
  └─ JSON을 우리가 보는 표로 포맷팅한다
```

CLI가 그냥 HTTP 클라이언트라는 걸 직접 증명할 수도 있다 — 데몬에 손으로 말 걸어보기:

```bash
# `docker ps`와 같은 결과, docker CLI는 안 씀
curl --unix-socket ~/.colima/default/docker.sock http://localhost/v1.43/containers/json
```

`docker ps`가 표로 만들어주는 그 컨테이너 데이터가 그대로 돌아온다.

## CLI는 어떤 데몬에게 말 걸지 어떻게 정하나

CLI는 *어떤* 데몬과 대화할지 다음 우선순위로 결정한다:

1. **`-H` / `--host` 또는 `--context` 플래그** — 명시적, 예: `docker -H tcp://1.2.3.4:2375 ps`
2. **`DOCKER_CONTEXT` 환경변수** — 설정하면 `DOCKER_HOST`와 활성 context를 모두 덮어쓴다
3. **활성 context** — `docker context ls` / `docker context use`
4. **`DOCKER_HOST` 환경변수** — 예: `unix:///path/to/docker.sock`
5. **기본값** — `/var/run/docker.sock`

Colima 환경에서는 Colima가 `~/.colima/default/docker.sock`을 가리키는 **context**를
등록(또는 `DOCKER_HOST` 설정)해 둔다. 그래서 플래그 없이도 CLI가 VM 안 데몬에 닿는다:

```bash
$ docker context ls
NAME       DESCRIPTION                  DOCKER ENDPOINT
colima *   colima                       unix:///Users/you/.colima/default/docker.sock
default    Current DOCKER_HOST based    unix:///var/run/docker.sock
```

`*`가 활성 context. context를 바꾸는 게 곧 같은 CLI를 (예: Docker Desktop으로)
다른 엔진에 재설치 없이 붙이는 방법이다.

## CLI가 말하는 Docker API

CLI와 데몬은 **Docker Engine API**로 통신한다 — HTTP로 말하는 버전 있는 REST API다.

- **버전 있음**: 요청에 버전이 붙는다(`/v1.43/...`). 최신 CLI도 협상으로 버전을
  낮춰 옛 데몬과 대화할 수 있다.
- **전송 방식 무관**: 같은 API가 Unix socket(로컬)으로도 TCP socket(원격)으로도
  돈다. CLI는 어느 쪽인지 신경 안 쓴다.
- **마법이 아님**: CLI가 하는 건 전부 socket에 `curl`로도 할 수 있다. CLI는 API
  위에 얹힌 편의 포맷터일 뿐이다.

## 이 슬롯을 채우는 다른 도구들

데몬이 표준 API를 노출하니까 클라이언트는 갈아끼울 수 있다:

| 클라이언트 | 비고 |
|---|---|
| `docker` | 표준 CLI. |
| `nerdctl` | **containerd**를 직접 다루는 Docker 호환 CLI. |
| `podman` | 데몬리스. CLI가 거의 Docker 호환 (`alias docker=podman`). |
| `crictl` | Kubernetes 디버깅에 쓰는 CRI 중심 CLI. |
| Docker SDK | 같은 API를 코드로 호출하는 Go/Python 등 라이브러리. |

## 정리

- CLI는 **얇은 클라이언트** — API 요청을 보내고 응답을 포맷팅한다.
- 엔진과 **분리**돼 있다. 하나의 CLI가 `--host`/`DOCKER_HOST`/context로 여러
  데몬을 겨냥할 수 있다.
- 처음의 "command not found" 버그는 엔진 문제가 아니라 **클라이언트 패키징/PATH**
  문제였다.
- 한 칸 아래: 이 API 요청을 실제로 받는 [dockerd, 데몬](./layer2_dockerd.md).

## 참고

- Docker Engine overview: <https://docs.docker.com/engine/>
- Docker CLI reference: <https://docs.docker.com/reference/cli/docker/>
- Docker Engine API: <https://docs.docker.com/engine/api/>
- nerdctl — containerd CLI: <https://github.com/containerd/nerdctl>
- crictl — CRI CLI: <https://github.com/kubernetes-sigs/cri-tools>
