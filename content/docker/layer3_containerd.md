+++
title = "Layer 3 — containerd (컨테이너 런타임)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "containerd", "runtime", "layers"]
+++

> Docker 레이어 시리즈 3편. 스택은 위에서 아래로:
> 1. docker CLI → 2. dockerd → **3. containerd** → 4. runc →
> 5. Linux 커널. [개요 글](./component_and_layers.md) 참고.

## 정체

`containerd`는 **컨테이너 런타임** — 위의 [dockerd](./layer2_dockerd.md)와 아래의
[runc](./layer4_runc.md) 사이에 앉은 데몬이다. 컨테이너 라이프사이클의 힘든 일을
도맡는다: 이미지 콘텐츠를 pull/저장하고, 파일시스템으로 풀어내고, 컨테이너를
시작부터 종료까지 감독한다.

CNCF **graduated 프로젝트**이자 업계 표준이다. 같은 `containerd`가 Docker, 대부분의
Kubernetes 클러스터, `nerdctl` 같은 도구들을 굴린다 — 애초에 Docker에서 분리해낸
이유가 바로 이거다.

## 왜 존재하나 (모놀리식 해체)

원래 Docker는 모놀리식이었다. Docker가 아키텍처를 모듈화하면서 런타임이 독립
프로젝트로 떨어져 나왔고(Docker 1.11, 2016), 이후 Kubernetes의 **CRI(Container
Runtime Interface)**가 이 분리를 더욱 강화했다. 다른 시스템들이 전체 Docker engine
**없이도** 런타임을 쓸 수 있게 하려고:

```
        dockerd   ── 필요 ──►  containerd  ◄── 필요 ──  Kubernetes (kubelet)
       (Docker)                  (공유)                  (CRI 경유)
```

그래서 `containerd`는 두 개의 API로 아주 다른 두 주인을 섬긴다:

- **dockerd**는 containerd의 **네이티브 gRPC API**로 말한다.
- **Kubernetes**는 **CRI**(Kubernetes가 정의한 gRPC API)로 말한다.

이 이중 역할 때문에 `dockerd`를 버려도(Kubernetes "dockershim 제거") 실제 런타임은
바뀌는 게 없다 — Kubernetes가 같은 `containerd`에 직접 말할 뿐이다.

## 무엇을 책임지나

`containerd`는 런타임 관심사를 소유한다 — `dockerd`보다 구체적이고 `runc`보다
추상적인 레이어:

- **이미지 관리**: 이미지 pull, digest 검증, **콘텐츠**를 content-addressable
  저장소에 보관, 스냅샷/레이어 관리.
- **스냅샷**: *snapshotter*(예: `overlayfs`)로 이미지 레이어에서 컨테이너 루트
  파일시스템을 준비.
- **컨테이너 라이프사이클**: create, start, pause, stop, delete — 그리고 실행
  프로세스를 **감독**해서 `dockerd`가 재시작돼도 살아남게 한다.
- **태스크(task)**: 컨테이너의 실제 실행 인스턴스.
- **shim**: 컨테이너마다 붙는 작은 `containerd-shim-runc-v2` 프로세스. 컨테이너의
  I/O와 종료 상태를 containerd 본체로부터 분리해 준다.

namespace/cgroup을 직접 만드는 일은 하지 않는다 — 그 마지막 단계는
[runc](./layer4_runc.md)에 위임한다.

## shim — containerd가 안전하게 재시작되는 이유

핵심 설계 포인트: `containerd`는 모든 컨테이너의 직접 부모로 남지 않는다. 대신
컨테이너마다 **shim**을 띄운다:

```
containerd
   └─ containerd-shim-runc-v2   ← 컨테이너 수명 내내 살아 있음
         └─ runc  (컨테이너를 만들려고 잠깐 돌고 종료)
               └─ 내 컨테이너 프로세스 (컨테이너 안에서 PID 1)
```

오래 사는 부모가 containerd가 아니라 shim이기 때문에:

- **`containerd`/`dockerd`를 재시작·업그레이드해도 돌고 있는 컨테이너가 안 죽는다**
  — shim이 살려두고 다시 연결한다.
- shim이 컨테이너의 **stdout/stderr**를 잡고 **종료 코드**를 containerd에 보고한다.

이게 `runc`가 `ps`에 거의 안 보이는 구조적 이유다: 할 일 하고 종료하면, shim이
지휘권을 넘겨받는다.

## 이 맥에서 보기

`containerd`는 Colima VM 안에서 돈다:

```bash
$ colima ssh -- sh -c 'command -v containerd; ctr --version'
/usr/bin/containerd
ctr github.com/containerd/containerd v1.7.x

# `ctr`는 containerd 자체 저수준 디버그 CLI (docker와 별개):
$ colima ssh -- sudo ctr namespaces ls
NAME    LABELS
default
moby            # <- dockerd가 containerd 안에서 쓰는 namespace
```

`moby`는 Docker가 쓰는 containerd namespace다 (Moby는 Docker의 업스트림 프로젝트
이름). `dockerd`가 그 아래에서 `containerd`를 부리고 있다는 걸 보여준다.

## containerd와 주변 레이어

| 관심사 | 소유자 |
|---|---|
| `docker build`, 네트워킹, 볼륨, Docker API | **dockerd** (레이어 2) |
| 이미지 pull/저장, 스냅샷, 라이프사이클, shim | **containerd** (레이어 3) |
| namespace/cgroup 생성, 프로세스 `exec` | **runc** (레이어 4) |

## 이 슬롯을 채우는 대안들

- **CRI-O** — *오직* Kubernetes CRI만을 위해 만든 런타임. 그 단일 목적엔 containerd
  보다 가볍고, Docker API는 없다.
- **다른 저수준 런타임을 쓰는 containerd** — containerd는 runtime-class 메커니즘으로
  `runc`, `crun`, gVisor(`runsc`), Kata를 골라 부릴 수 있다
  ([레이어 4](./layer4_runc.md) 참고).

## 정리

- `containerd`는 **표준 런타임** — 이미지 콘텐츠, 스냅샷, 컨테이너 라이프사이클.
- Kubernetes 등이 CRI로 직접 쓸 수 있게 **Docker에서 분리**됐고, `dockerd`를
  우회한다.
- **shim** 설계 덕에 데몬을 재시작해도 컨테이너가 안 멈춘다.
- 마지막 저수준 컨테이너 생성은 여전히 [runc](./layer4_runc.md)에 위임한다
  (다음 레이어).

## 참고

- containerd 공식 사이트: <https://containerd.io/>
- containerd GitHub: <https://github.com/containerd/containerd>
- CNCF containerd graduation 발표: <https://www.cncf.io/announcements/2019/02/28/cncf-announces-containerd-graduation/>
- Docker — "What is containerd?": <https://www.docker.com/blog/what-is-containerd-runtime/>
- CRI-O: <https://cri-o.io/>
