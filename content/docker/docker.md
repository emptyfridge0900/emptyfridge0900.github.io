+++
title="Docker? Dockerd? Containerd?"
date=2025-04-21

[taxonomies]
categories = ["post"]
tags = ["docker"]
+++

I had been using Docker for a while, but I had never really studied it. I mostly knew it
as something where I copied commands from the internet and ran them. So I spent some time
figuring out what Docker actually does, and why Docker did not run on my Mac.

### What is Docker?

Docker is a [container-based open-source virtualization platform](https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html).
So Docker is a platform: a container-based platform, a virtualization platform, and an
open-source platform.

Docker-related projects include Docker Compose, private registry, and Docker for Desktop,
but the main component is still Docker Engine.

When I install Docker on a Mac and try to run it, I see:

```text
Cannot connect to the Docker daemon... Is the docker daemon running?
```

It complains about the Docker daemon. But I definitely installed Docker, so what does it
mean that the Docker daemon is not running?

### What is the Docker daemon?

[Docker Engine](https://docs.docker.com/engine/) consists of a server, an API, and a
client. **dockerd** is the daemon used by the server, and **docker** is the client.

So did I install only the Docker client part? First, I tried to run the Docker daemon.

There are [two ways to start the Docker daemon](https://docs.docker.com/engine/daemon/start/):
starting it as a service, or running it in the foreground with the `dockerd` command. For
daemon configuration and startup details, I referred to [this post](https://junstar92.tistory.com/169).

Because I use a Mac, `systemctl` did not work and the `dockerd` command did not work
either, so I looked for another approach.

### colima

When trying to use Docker on a Mac, I often see articles recommending Podman or Colima as
alternatives to Docker Desktop. I wondered why Docker Desktop, Podman, and Colima kept
appearing while I was searching for how to run the Docker daemon.

Docker Desktop includes both `dockerd` and `docker`. Colima is an alternative to Docker
Desktop that runs `dockerd` and `docker`-compatible runtime support for me.

In short, Colima is a helper that runs a container runtime, or Docker runtime. If Colima
is installed, I can start the Docker runtime simply with:

```bash
colima start
```

### lima

Colima is based on Lima. The previously mentioned Podman also uses Lima underneath.

The [Lima homepage](https://lima-vm.io/) defines Lima like this:

> Lima launches Linux virtual machines with automatic file sharing and port forwarding (similar to WSL2).

A Linux VM... that sounds like the container side of the story.

In the documentation's [motivation section](https://lima-vm.io/docs/#motivation), Lima
says one of its goals is to promote containerd.

`containerd` was a new term to me, so I looked up [containerd](https://containerd.io/).
It is a container runtime.

containerd was originally part of Docker, but with the introduction of CRI, it was split
out of Docker into an independent project. Along with CRI-O, it became one of the most
commonly used container runtimes.

In one sentence: Docker was originally a monolithic program, but Kubernetes needed a
cleaner way to use Docker's container runtime, so Docker was split into several tools,
including containerd.

I wanted to study Docker Engine more deeply, but I had already spent more than two hours
getting this far. The rest can wait.

### Three-line summary

1. Docker Engine originally handled everything by itself.
2. With the introduction of CRI, containerd was separated from Docker.
3. Docker Desktop is fine, but on a Mac you can also use Lima or Colima to run the Docker daemon.

### References

* <https://junstar92.tistory.com/169>
* <https://www.docker.com/blog/containerd-vs-docker/>
* <https://kr.linkedin.com/pulse/containerd%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9D%B4%EA%B3%A0-%EC%99%9C-%EC%A4%91%EC%9A%94%ED%95%A0%EA%B9%8C-sean-lee>
* <https://blog.siner.io/2021/10/23/container-ecosystem/>
