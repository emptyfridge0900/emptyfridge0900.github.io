+++
title = "Layer 2 — dockerd (데몬 / API 서버)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "dockerd", "daemon", "layers"]
+++

> Docker 레이어 시리즈 2편. 스택은 위에서 아래로:
> 1. docker CLI → **2. dockerd (데몬)** → 3. containerd → 4. runc →
> 5. Linux 커널. [개요 글](./component_and_layers.md) 참고.

## 정체

`dockerd`는 [**Docker 데몬**](https://docs.docker.com/get-started/docker-overview/#docker-architecture) —
사람들이 "Docker"라고 하면 흔히 떠올리는 그 오래 떠 있는 서버 프로세스다. [docker CLI](./layer1_docker_cli.md)는 클라이언트일
뿐이고, `dockerd`가 실제 관리 일을 하는 서버다. socket에서 듣고 있다가 Docker API
요청을 받아, 그걸 처리하는 데 필요한 모든 걸 조율한다.

맥에서 흔히 보는 ``Cannot connect to the Docker daemon... Is the docker daemon
running?`` 에러의 주인공이 바로 이 프로세스다 — CLI가 socket에는 닿았지만 듣고
있는 `dockerd`가 없다는 뜻이다 (맥에서는 그걸 돌릴 VM이 안 떠 있어서).

## 무엇을 책임지나

`dockerd`는 "컨테이너를 날것으로 실행하는 행위"를 **제외한** 모든 "Docker 레벨"
개념을 소유한다:

- **이미지**: 레지스트리에서 pull, 빌드(`docker build`), 태깅, 레이어 저장,
  빌드 캐시 관리.
- **네트워킹**: bridge 네트워크 생성, 포트 매핑(`-p 8080:80`), 컨테이너 간 DNS,
  기본 `bridge`/`host`/`none` 네트워크.
- **볼륨 & 스토리지**: named volume, bind mount, 이미지 레이어를 쌓는 스토리지
  드라이버(`overlay2`).
- **API 표면**: 전체 Docker Engine API를 클라이언트에 제공.
- **고수준 객체**: `docker compose`(데몬 경유), swarm 모드, 플러그인.

정작 컨테이너의 저수준 실행 자체는 직접 하지 않는다 —
[containerd](./layer3_containerd.md)로 위임한다.

## "Docker engine = server + api + client"

Docker **Engine**은 세 가지다:

```
Docker Engine
├── client   →  docker (CLI)             [레이어 1]
├── api      →  Docker Engine REST API (둘 사이의 계약)
└── server   →  dockerd (데몬)           [레이어 2]  ← 이 글
```

그래서 맥에서 "Docker를 깔았는데 안 돌아간다"는 보통: **클라이언트**는 깔렸는데
**서버**(`dockerd`)가 돌 곳이 없다는 뜻이다. `dockerd`는 Linux 프로세스인데
맥은 Linux가 아니니까.

## 왜 맥에서 네이티브로 못 도나

[`dockerd`는 Linux 또는 Windows에서 도는 프로그램이다](https://docs.docker.com/reference/cli/dockerd/).
Linux에서는 커널 기능(namespace, cgroup, overlay 파일시스템)에, Windows에서는
HCS(Host Compute Service)에 의존한다. macOS(Darwin) 커널 위에서는 돌 수 없다. 이게
Docker Desktop / Colima / Lima가 존재하는 이유 전부다: 얘네가 **Linux VM**을
띄우고 그 *안에서* `dockerd`를 돌린다. 그럼 내 맥의 CLI가 포워딩된 socket을 통해
VM 안으로 손을 뻗는 것이다.

```bash
# dockerd는 맥이 아니라 Colima VM 안에 산다:
$ colima ssh -- sh -c 'command -v dockerd; ps -e -o comm | grep dockerd'
/usr/bin/dockerd
dockerd
```

## 모놀리식 해체: dockerd는 아래로 위임한다

Docker는 원래 **모놀리식**이었다 — 한 프로그램이 API 서빙, 이미지 관리, 실제
컨테이너 실행을 다 했다. 시간이 지나며 쪼개졌다:

```
   예전 (모놀리식)                   지금
   ┌──────────────┐                  ┌──────────────┐  dockerd: API, 이미지,
   │              │                  │   dockerd    │  네트워킹, 볼륨
   │    docker    │                  └──────┬───────┘
   │   (전부 다)   │        →                │ gRPC
   │              │                  ┌──────▼───────┐  containerd: 런타임,
   │              │                  │  containerd  │  라이프사이클, 이미지 콘텐츠
   └──────────────┘                  └──────┬───────┘
                                            │
                                     ┌──────▼───────┐  runc: 컨테이너 생성하고
                                     │     runc     │  바로 종료
                                     └──────────────┘
```

그래서 `docker run`을 하면 `dockerd`가 이미지를 해석하고 네트워킹·볼륨을 세팅한
뒤, 로컬 [gRPC API](https://docs.docker.com/reference/cli/dockerd/)로
**containerd**를 불러 실제로 컨테이너를 시작시킨다. `dockerd`는
계속 떠서 전체를 관리하고, 실행 자체는 위임한다.

이 분리 덕에 **Kubernetes가 `dockerd`를 버릴 수 있었다**
("[dockershim 제거](https://kubernetes.io/blog/2022/05/03/dockershim-historical-context/)"):
Docker Engine은 [CRI](https://kubernetes.io/docs/concepts/containers/cri/)를
네이티브로 구현하지 않아서 Kubernetes가 dockershim이라는 브릿지를 유지해야 했다.
그 부담을 없애기 위해 Kubernetes는 CRI를 직접 구현하는 `containerd`에 말하고
`dockerd` 레이어를 통째로 건너뛴다.

## 데몬 설정

`dockerd`의 동작은 [`daemon.json`](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file)으로
제어한다 (Colima에서는 VM 안에 있다):

```jsonc
// /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://..."],
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

일반 Linux 호스트에서는 systemd가 `dockerd`를 띄우거나(`systemctl start docker`),
디버깅용으로 포그라운드로 직접 돌린다(`sudo dockerd`). Colima에서는 직접 띄울 일이
없다 — `colima start`가 VM을 부팅하면 VM이 알아서 `dockerd`를 띄운다.

## 이 슬롯을 채우는 대안들

| 도구 | dockerd와 다른 점 |
|---|---|
| [**Podman**](https://docs.podman.io/en/stable/markdown/podman.1.html) | **데몬리스** — 항상 떠 있는 서버 없이 컨테이너를 자식 프로세스로 실행. 기본 rootless. |
| [**CRI-O**](https://cri-o.io/) | Kubernetes [CRI](https://kubernetes.io/docs/concepts/containers/cri/) 전용 최소 런타임. Docker API도 `build`도 없음. |
| **containerd (단독)** | `dockerd` 없이도 사용 가능 (예: `nerdctl`). |

## 정리

- `dockerd`는 **서버** — 이미지·네트워킹·볼륨, 그리고 API를 소유한다.
- "Docker daemon not running" = 서버가 안 떠 있다는 뜻 (맥에서는 VM이 안 떠 있는 것).
- 컨테이너를 가장 낮은 레벨에서 돌리는 건 얘가 **아니다** —
  [containerd](./layer3_containerd.md)로 **위임**한다 (다음 레이어).
- Kubernetes는 이 레이어를 통째로 건너뛰고 containerd에 직접 말한다.

## 참고

- Docker 아키텍처 — dockerd가 이미지·네트워크·볼륨을 소유하는 구조: <https://docs.docker.com/get-started/docker-overview/#docker-architecture>
- `dockerd` 레퍼런스 — `--containerd` gRPC 주소, daemon.json 설정, Linux/Windows 의존성: <https://docs.docker.com/reference/cli/dockerd/>
- daemon.json 설정 (storage-driver, log-driver 등): <https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file>
- Dockershim 제거의 역사적 배경 — Docker가 CRI를 구현하지 않은 이유: <https://kubernetes.io/blog/2022/05/03/dockershim-historical-context/>
- Dockershim 제거 FAQ: <https://kubernetes.io/blog/2022/02/17/dockershim-faq/>
- Kubernetes Container Runtime Interface(CRI) 스펙: <https://kubernetes.io/docs/concepts/containers/cri/>
- Kubernetes 컨테이너 런타임 설정 (containerd, CRI-O): <https://kubernetes.io/docs/setup/production-environment/container-runtimes/>
- Podman — 데몬리스 OCI 컨테이너 엔진: <https://docs.podman.io/en/stable/markdown/podman.1.html>
