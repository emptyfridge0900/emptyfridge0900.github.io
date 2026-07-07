+++
title="Docker? dockerd? containerd?"
date=2025-04-21

[taxonomies]
categories = ["post"]
tags = ["docker"]
+++

그동안 Docker를 계속 사용해 왔지만, Docker 자체를 제대로 공부한 적은 없었다. 인터넷에 떠도는 명령어를 따라 치는 정도로만 알고 사용했다. 그래서 Docker가 정확히 무엇을 하는 도구인지, 그리고 왜 내 Mac에서 Docker가 바로 동작하지 않는지 공부해 보았다.

### Docker란 무엇인가?
Docker는 [container 기반의 open-source virtualization platform](https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html)이라고 설명할 수 있다.

즉 Docker는 platform이고, container 기반 platform이며, virtualization platform이고, open-source platform이다. Docker 생태계에는 Docker Compose, private registry, Docker Desktop 같은 여러 프로젝트가 있지만, 중심이 되는 것은 Docker Engine이다.

Mac에서 `docker`를 설치한 뒤 실행해 보면 다음과 같은 메시지를 만날 수 있다.

```text
Cannot connect to the Docker daemon... Is the docker daemon running?
```

분명 Docker를 설치했는데 Docker daemon이 돌고 있지 않다는 말은 무엇일까?

### 그럼 docker daemon이라는 녀석은 뭘까?
[Docker Engine](https://docs.docker.com/engine/)은 크게 server, API, client로 구성된다. **dockerd**는 server 쪽 daemon의 이름이고, **docker**는 CLI client다.

그렇다면 내가 설치한 것은 Docker client뿐이었던 걸까? 일단 Docker daemon을 실행해야 한다.

[Docker daemon을 실행하는 방법](https://docs.docker.com/engine/daemon/start/)은 보통 두 가지로 설명된다. service로 시작하는 방법과 `dockerd` 명령어로 foreground에서 실행하는 방법이다. daemon 설정과 실행 방법은 [여기](https://junstar92.tistory.com/169)를 참고할 수 있다.

하지만 나는 Mac을 사용하고 있어서 `systemctl`도 안 되고, `dockerd` 명령어도 바로 먹히지 않았다. 그래서 다른 방법을 찾아보았다.

### Colima
Mac에서 Docker를 쓰려고 찾아보면 Docker Desktop의 대체재로 Podman이나 Colima를 사용하라는 글을 자주 보게 된다. Docker daemon을 실행하는 방법을 찾고 있었는데, 왜 갑자기 Docker Desktop, Podman, Colima가 나오는지 궁금했다.

Docker Desktop에는 `dockerd`와 `docker` CLI가 함께 포함되어 있다. Colima는 Docker Desktop의 대체재로, Mac 위에서 Docker runtime이 돌아갈 Linux VM을 준비하고 Docker daemon에 접근할 수 있게 해준다.

간단히 말하면 Colima는 container runtime을 실행할 수 있는 local Linux VM 환경을 만들어 주는 도구다. Colima가 설치되어 있다면 다음 명령으로 Docker runtime을 쉽게 시작할 수 있다.

```bash
colima start
```

### Lima
Colima는 Lima를 기반으로 하는 프로그램이다. 앞서 언급한 Podman도 Mac에서는 Lima 기반 환경을 사용할 수 있다.

Lima [홈페이지](https://lima-vm.io/)에서는 Lima를 이렇게 설명한다.

> Lima launches Linux virtual machines with automatic file sharing and port forwarding (similar to WSL2).

즉 Lima는 Mac에서 Linux VM을 띄우고, file sharing과 port forwarding을 자동으로 설정해 주는 도구다. container 자체라기보다는 container runtime을 실행할 수 있는 Linux 환경을 제공하는 쪽에 가깝다.

또 [motivation](https://lima-vm.io/docs/#motivation)을 보면 Lima의 목표 중 하나가 containerd를 promote하는 것이라고 한다.

`containerd`는 처음 들어보는 단어라서 [containerd](https://containerd.io/)가 무엇인지 또 찾아보았다. containerd는 container runtime이다.

containerd는 원래 Docker의 일부였지만, Kubernetes의 CRI(Container Runtime Interface) 흐름과 함께 Docker에서 분리되어 독립적인 프로젝트가 되었다. 지금은 CRI-O와 함께 자주 언급되는 대표적인 container runtime이다.

한마디로 Docker는 원래 비교적 monolithic한 프로그램이었지만, Kubernetes와 container 생태계가 발전하면서 역할이 쪼개졌고, 그 결과 containerd 같은 runtime component가 독립적으로 중요해졌다고 볼 수 있다.

Docker Engine을 더 공부해 보려고 했는데 여기까지 정리하는 데만 벌써 2시간이 넘게 지났다. 나머지는 나중에 더 공부하자.

### 3줄요약?
1. Docker는 원래 Docker Engine이 많은 일을 한 번에 담당하는 구조였다.
2. CRI 흐름과 container 생태계 변화 속에서 containerd가 Docker에서 분리되어 중요한 runtime component가 되었다.
3. Mac에서는 Docker Desktop을 써도 되고, Lima/Colima를 사용해 Docker daemon이 돌아갈 Linux VM 환경을 만들 수도 있다.


### 참고
* https://junstar92.tistory.com/169
* https://www.docker.com/blog/containerd-vs-docker/
* https://kr.linkedin.com/pulse/containerd%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9D%B4%EA%B3%A0-%EC%99%9C-%EC%A4%91%EC%9A%94%ED%95%A0%EA%B9%8C-sean-lee
* https://blog.siner.io/2021/10/23/container-ecosystem/
