+++
title = "Layer 4 — runc (저수준 OCI 런타임)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "runc", "oci", "layers"]
+++

> Docker 레이어 시리즈 4편. 스택은 위에서 아래로:
> 1. docker CLI → 2. dockerd → 3. containerd → **4. runc** →
> 5. Linux 커널. [개요 글](./component_and_layers.md) 참고.

## 정체

`runc`는 **저수준 런타임** — *실제로 컨테이너를 생성하는* 프로그램이다. 그 위의
모든 것(CLI, dockerd, containerd)은 오케스트레이션이고, `runc`는 컨테이너가
추상 개념이기를 멈추고 [Linux 커널](./component_and_layers.md) 위의 진짜 격리된
프로세스가 되는 지점이다.

작고, 한 가지 일만 하고, 컨테이너를 시작시킨 뒤 **즉시 종료**한다.
[**OCI Runtime Spec**](https://github.com/opencontainers/runtime-spec)의
[레퍼런스 구현체](https://github.com/opencontainers/runtime-spec/blob/main/implementations.md)다.

## "컨테이너를 생성한다"는 게 실제로 무슨 뜻인가

컨테이너는 가상머신이 아니다. 그냥 **주변 세상에 대해 속고 있는 평범한 Linux
프로세스**다. `runc`가 Linux 커널의 세 가지 기능으로 그 거짓말을 한다:

- [**namespace**](https://man7.org/linux/man-pages/man7/namespaces.7.html) —
  프로세스가 *무엇을 볼 수 있는지*를 격리한다.
  [OCI spec이 정의하는 8종](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#namespaces):
  PID(자기만의 PID 1), 마운트(자기만의 파일시스템 뷰), 네트워크(자기만의 인터페이스),
  사용자, 호스트명(UTS), IPC, cgroup(자기만의 cgroup 뷰), time(자기만의 시계)을
  각각 분리한다.
- [**cgroup**](https://man7.org/linux/man-pages/man7/cgroups.7.html)(control group) —
  프로세스가 *무엇을 얼마나 쓸 수 있는지*를
  [제한한다](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#control-groups):
  CPU, 메모리, PID 수, I/O. `--memory=512m`이 강제되는 방식이 이거다.
- **루트 파일시스템(rootfs)** — `runc`가 containerd가 이미지 레이어로 준비해 둔
  파일시스템으로 `pivot_root` 해서, 프로세스가 호스트가 아니라 이미지의 파일을
  `/`로 보게 만든다.

여기에 보안 레이어도 적용한다:
[**capabilities**](https://github.com/opencontainers/runtime-spec/blob/main/config.md)(root 권한 떨구기),
[**seccomp**](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#seccomp)(syscall 필터),
옵션으로 AppArmor/SELinux. 그런 다음 그 만들어진 세상 안에서
컨테이너의 entrypoint를 PID 1로 `exec`하고 — 종료한다.

## 왜 `ps`에 거의 안 나오나

`runc`는 **오래 살지 않는다**. 컨테이너 *생성* 동안만 돌고,
[shim](./layer3_containerd.md)에게 넘긴 뒤 종료한다. 그래서:

```bash
$ colima ssh -- sh -c 'ps -e -o comm | grep -E "dockerd|containerd|runc"'
dockerd
containerd
containerd-shim-runc-v2     # 오래 사는 부모
# runc는 보통 여기 없다 — 이미 종료됨
```

`runc`를 프로세스 목록에서 잡으려면 시작이 느린 무언가를 `docker run` 하는 *도중에*
봐야 한다. 그 후엔 컨테이너 프로세스가 shim에 reparent되고, `runc`는 사라진다.
이건 의도된 설계다 ([레이어 3](./layer3_containerd.md)의 shim 설명 참고).

## OCI Runtime Spec — runc가 교체 가능한 이유

`runc`는 그저 **Open Container Initiative(OCI) Runtime Specification**의 *레퍼런스*
구현일 뿐이다. 이 spec은 표준
["bundle"](https://github.com/opencontainers/runtime-spec/blob/main/bundle.md)(rootfs + `config.json`)과
런타임이 지원해야 하는
[명령들](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md)(`create`, `start`, `kill`, `delete`)을
정의한다.
[containerd](./layer3_containerd.md)가 이 spec으로 말하므로, OCI를 따르는 어떤
런타임이든 갈아끼울 수 있다:

| 저수준 런타임 | 바뀌는 점 |
|---|---|
| [**runc**](https://github.com/opencontainers/runc) | 기본값. 표준 Linux namespace + cgroup. Go로 작성. |
| [**crun**](https://github.com/containers/crun) | 같은 모델, C로 작성 — 시작 빠르고 메모리 적음. |
| [**gVisor (`runsc`)**](https://gvisor.dev/docs/architecture_guide/security/) | 실제 커널 앞에 **유저 공간 커널**을 둔다 — 격리 강함, 약간의 오버헤드. |
| [**Kata Containers**](https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md) | 각 컨테이너를 **경량 마이크로 VM**에서 실행 — VM급 격리 + 컨테이너 사용성. |
| [**youki**](https://github.com/youki-dev/youki) | Rust로 작성한 runc 호환 런타임. |

모두 같은 OCI 인터페이스를 구현하므로, (예: 더 강한 격리를 위해) 런타임을 바꿔도
CLI·dockerd·containerd는 **건드릴 필요가 없다** — runtime-class 설정만 바꾸면 된다.

## runc가 소비하는 spec 보기

VM 안에서 runc를 직접 들여다볼 수 있다:

```bash
$ colima ssh -- sh -c 'command -v runc; runc --version'
/usr/bin/runc
runc version 1.x.x
spec: 1.x.x          # <- 구현하는 OCI Runtime Spec 버전

# runc는 "bundle"을 다룬다: rootfs 디렉터리 + config.json
# config.json이 namespace, cgroup, 마운트, capabilities, 프로세스를 기술한다
```

그 `config.json`이 바로 containerd가 생성해서 `runc`에 건네는 것이고, `runc`는
그걸 읽어 거기 적힌 그대로의 컨테이너를 만든다.

## 전체 핸드오프 흐름

```
docker run nginx
  └─ dockerd                    [레이어 2] 이미지 해석, 네트워크/볼륨 세팅
       └─ containerd            [레이어 3] 이미지 → rootfs 풀기, "task" 생성
            └─ containerd-shim-runc-v2    (상주 프로세스)
                 └─ runc        [레이어 4] namespace/cgroup, pivot_root, exec → 종료
                      └─ nginx  [       ] 컨테이너 안 PID 1
                           └─   [레이어 5] Linux 커널
```

## 정리

- `runc`는 컨테이너가 **진짜 격리된 Linux 프로세스**가 되는 곳이다 —
  namespace(보는 것), cgroup(쓰는 것), rootfs(파일시스템).
- **수명이 짧다**: 컨테이너를 만들고 shim에 넘긴 뒤 종료한다 — 그래서 `ps`에 보통
  안 보인다.
- **OCI Runtime Spec**을 구현하므로 위 레이어를 건드리지 않고 **교체 가능**하다
  (crun, gVisor, Kata, youki).
- `runc`를 쓸 때 그 아래의 **Linux 커널**(레이어 5)은 갈아끼울 수 없는 토대다.
  그 namespace와 cgroup이 이 모든 걸 가능하게 한다. 단, gVisor는 유저 공간 커널로,
  Kata는 게스트 커널로 이 토대를 추상화할 수 있다.

## 참고

- runc — OCI Runtime Spec 레퍼런스 구현 (seccomp, AppArmor, SELinux 포함): <https://github.com/opencontainers/runc>
- OCI Runtime Spec 구현체 목록 — runc가 레퍼런스: <https://github.com/opencontainers/runtime-spec/blob/main/implementations.md>
- OCI Runtime Spec — namespace 8종 정의 (PID, mount, network, user, UTS, IPC, cgroup, time): <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#namespaces>
- OCI Runtime Spec — cgroup 리소스 제한 (CPU, memory, PIDs, Block IO): <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#control-groups>
- OCI Runtime Spec — capabilities 정의: <https://github.com/opencontainers/runtime-spec/blob/main/config.md>
- OCI Runtime Spec — seccomp 필터 설정: <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#seccomp>
- OCI Runtime Spec — bundle 포맷 (rootfs + config.json): <https://github.com/opencontainers/runtime-spec/blob/main/bundle.md>
- OCI Runtime Spec — 라이프사이클 (create, start, kill, delete): <https://github.com/opencontainers/runtime-spec/blob/main/runtime.md>
- Linux namespaces(7) man page: <https://man7.org/linux/man-pages/man7/namespaces.7.html>
- cgroups(7) man page: <https://man7.org/linux/man-pages/man7/cgroups.7.html>
- crun — C로 작성, runc 대비 49.4% 빠른 시작: <https://github.com/containers/crun>
- gVisor 보안 아키텍처 — 유저 공간 커널(Sentry)로 syscall 가로채기: <https://gvisor.dev/docs/architecture_guide/security/>
- Kata Containers 아키텍처 — 경량 VM 기반 컨테이너 격리: <https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md>
- youki — Rust로 작성한 OCI 런타임: <https://github.com/youki-dev/youki>
