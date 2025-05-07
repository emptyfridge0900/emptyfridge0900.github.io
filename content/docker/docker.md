+++
title="Docker? Dockerd? Containerd?"
date=2025-04-21

[taxonomies]
categories = ["post"]
tags = ["docker"]
+++

그동안 도커를 사용해왔지만 도커에 대해 공부해본적 없이 그냥 인터넷에 떠도는 명령어 따라 치는 정도로만 알고 사용했다. 그래서 도커가 뭐하는 녀석인지 그리고 왜 내 맥에서 도커가 안돌아가는지 공부를 좀 해보았다.
### 도커란무엇인가?
도커는 [컨테이너 기반의 오픈소스 가상화 플렛폼이다](https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html).  
그러니까 도커는 플랫폼이고,  컨테이너 기반 플랫폼,  가상화 플랫폼,  오픈소스 플랫폼인 것이다.  
도커 관련 프로젝트는 docker compose, private registry, docker for desktop 등이 있다 하지만 메인은 역시 docker engine이다.

일단 mac에서 docker를 설치하고 docker를 실행해보려고 하면 'Cannot connect to the Docker daemon...Is the docker daemon running?'  
라면서 docker daemon 타령을 하는데... 난 분명 도커를 설치했는데 docker daemon이 안돌고 있다는 말은 뭘까?  

### 그럼 docker daemon이라는 녀석은 뭘까?
[docker engine](https://docs.docker.com/engine/)은 서버, api, 클라이언트로 되어있다. **dockerd**는 서버에서 사용하는 daemon의 이름이고 **docker**는 클라이언트다.  
그러니까 나는 도커 클라이언트 부분만 설치했다는건가🤔? 일단 docker daemon을 실행해보자.  
[docker deamon을 실행하는 방법에는 두가지가 있다고 하는데](https://docs.docker.com/engine/daemon/start/),  서비스로 시작하는 방법과 dockerd 명령어로 foreground로 실행하는 방법이다.  
docker deamon을 설정하고 실행하는 방법은  [여기](https://junstar92.tistory.com/169)를 참고해보자.
  
나는 mac을 사용하고 있어서 그런지 systemctl도 안되고 dockerd 명령어도 안먹혀서 다른 방법을 찾아보았다.

### colima
mac에서 docker를 쓰려고 하다보면 docker desktop 의 대체체로 podman, colima를 사용하라는 글을 자주 볼수 있다.
docker deamon을 실행시키는 방법을 찾는데 뜬금없이 docker desktop, podmna, colima 등이 난데없이 튀어나오는지 궁금했다.
  
Docker desktop에는 dockerd와 docker 둘다 포함되있는데 colima가 docker desktop의 대체품으로 dockerd랑 docker를 실행시켜주는 것이다.  
colima는 간단히 말해서 container runtime (docker runtime)을 실행해주는 대리이다.  
colima가 설치되어있다면 colima start로 docker runtime을  간단하게 시작할 수 있다.    



### lima
colima는 lima를 base로 하는 프로그램이다. 앞서 언급한 podman도 lima를 베이스로 한다.  
lima [홈페이지](https://lima-vm.io/)에서 lima의 정의를 보면 
_Lima launches Linux virtual machines with automatic file sharing and port forwarding (similar to WSL2)._  
라고 되어있다. 리눅스 vm 이라... 컨테이너를 말하는거 같다.
그리고 Documentation 에서 [motivation](https://lima-vm.io/docs/#motivation)을 보면 promote containerd가 lima의 목표라고 한다.  
containerd는 처음들어보는  단어다  그래서 [containerd](https://containerd.io/ )가 뭔지를 또 찾아보았다.  이녀석은 컨테이너의 런타임이다.  
containerd는 원래 docker의 일부였으나 cri 도입으로 docker에서 분리되어 독자적인 프로젝트가 되었고 cri-o와 함께 가장 자주 쓰이는 컨테이너 런타임이 되었다고 한다.  
한마디로 docker는 원래 monolithic한 프로그램이였지만, 쿠버네티가 docker의 컨테이너 런타임을 쉽게 쓸 수있게 해주려고 docker를 오체분시 시켜서 containerd를 비롯한 여러 툴들로 나누었다는 말이다.  

도커엔진에 대해 더 공부해보려고 했는데 여기까지 공부하는데 시간이 벌써 2시간 넘게 지났다. 나머지는 나중에 공부하자.  

### 3줄요약?
1. 도커는 원래 docker engine 혼자 모든 일을 다 맡아서 했음.
2. cri 도입하면서 containerd를 docker에서 분리해냄.
3. docker desktop써도 되지만 mac에서는 lima, colima 등을 사용하여 docker demon을 실행시킬수 있다.


### 참고
* https://junstar92.tistory.com/169
* https://www.docker.com/blog/containerd-vs-docker/
* https://kr.linkedin.com/pulse/containerd%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9D%B4%EA%B3%A0-%EC%99%9C-%EC%A4%91%EC%9A%94%ED%95%A0%EA%B9%8C-sean-lee
* https://blog.siner.io/2021/10/23/container-ecosystem/