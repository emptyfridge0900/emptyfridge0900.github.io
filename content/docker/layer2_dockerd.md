+++
title = "Layer 2 — dockerd (Daemon / API Server)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "dockerd", "daemon", "layers"]
+++

> Part 2 of the Docker layers series. From top to bottom, the stack is:
> 1. docker CLI → **2. dockerd (daemon)** → 3. containerd → 4. runc →
> 5. Linux kernel. See the [overview article](./component_and_layers.md).

## Identity

`dockerd` is the [**Docker daemon**](https://docs.docker.com/get-started/docker-overview/#docker-architecture):
the long-running server process people usually imagine when they say "Docker." The
[docker CLI](./layer1_docker_cli.md) is only the client. `dockerd` is the server that
does the actual management work. It listens on a socket, receives Docker API requests,
and coordinates everything needed to handle them.

The familiar macOS error ``Cannot connect to the Docker daemon... Is the docker daemon
running?`` points to this process. It means the CLI reached the socket path, but no
`dockerd` was listening there, usually because the VM that runs it on a Mac is not up.

## What it owns

`dockerd` owns every "Docker-level" concept **except** the raw act of running a
container:

- **Images**: pulling from registries, building with `docker build`, tagging, layer storage, and build cache management.
- **Networking**: bridge network creation, port mapping (`-p 8080:80`), DNS between containers, and the default `bridge`/`host`/`none` networks.
- **Volumes and storage**: named volumes, bind mounts, and storage drivers such as `overlay2` that stack image layers.
- **API surface**: the full Docker Engine API exposed to clients.
- **High-level objects**: `docker compose` through the daemon, swarm mode, and plugins.

It does not directly perform the low-level container execution itself. It delegates that
to [containerd](./layer3_containerd.md).

## "Docker engine = server + api + client"

Docker **Engine** consists of three parts:

```text
Docker Engine
├── client   →  docker (CLI)             [Layer 1]
├── api      →  Docker Engine REST API (contract between them)
└── server   →  dockerd (daemon)         [Layer 2]  ← this article
```

So on a Mac, "I installed Docker, but it does not run" usually means: the **client** is
installed, but there is nowhere for the **server** (`dockerd`) to run. `dockerd` is a
Linux process, and macOS is not Linux.

## Why it cannot run natively on macOS

[`dockerd` runs on Linux or Windows](https://docs.docker.com/reference/cli/dockerd/).
On Linux it depends on kernel features such as namespaces, cgroups, and overlay
filesystems. On Windows it depends on HCS, the Host Compute Service. It cannot run on top
of the macOS Darwin kernel. This is the whole reason Docker Desktop, Colima, and Lima
exist: they start a **Linux VM** and run `dockerd` *inside* it. Then the CLI on the Mac
reaches into the VM through a forwarded socket.

```bash
# dockerd lives inside the Colima VM, not on the Mac:
$ colima ssh -- sh -c 'command -v dockerd; ps -e -o comm | grep dockerd'
/usr/bin/dockerd
dockerd
```

## Breaking up the monolith: dockerd delegates downward

Docker was originally **monolithic**: one program served the API, managed images, and ran
containers. Over time, it was split apart:

```text
   Old (monolithic)                 Now
   ┌──────────────┐                  ┌──────────────┐  dockerd: API, images,
   │              │                  │   dockerd    │  networking, volumes
   │    docker    │                  └──────┬───────┘
   │  (all of it) │        →                │ gRPC
   │              │                  ┌──────▼───────┐  containerd: runtime,
   │              │                  │  containerd  │  lifecycle, image content
   └──────────────┘                  └──────┬───────┘
                                            │
                                     ┌──────▼───────┐  runc: creates container
                                     │     runc     │  and exits immediately
                                     └──────────────┘
```

So when you run `docker run`, `dockerd` interprets the image and sets up networking and
volumes. Then it calls **containerd** through a local
[gRPC API](https://docs.docker.com/reference/cli/dockerd/) to actually start the
container. `dockerd` stays alive to manage the whole system, while execution itself is
delegated.

This separation is what allowed **Kubernetes to drop `dockerd`**
("[dockershim removal](https://kubernetes.io/blog/2022/05/03/dockershim-historical-context/)").
Docker Engine did not implement the
[CRI](https://kubernetes.io/docs/concepts/containers/cri/) natively, so Kubernetes had to
maintain a bridge called dockershim. To remove that burden, Kubernetes talks directly to
`containerd`, which implements CRI, and skips the entire `dockerd` layer.

## Daemon configuration

`dockerd` behavior is controlled through
[`daemon.json`](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file),
which lives inside the VM when using Colima:

```jsonc
// /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://..."],
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

On a normal Linux host, systemd starts `dockerd` (`systemctl start docker`), or you can
run it in the foreground for debugging (`sudo dockerd`). In Colima, you normally do not
start it yourself. `colima start` boots the VM, and the VM starts `dockerd`.

## Alternatives that fill this slot

| Tool | Difference from dockerd |
|---|---|
| [**Podman**](https://docs.podman.io/en/stable/markdown/podman.1.html) | **Daemonless**: runs containers as child processes without an always-on server. Rootless by default. |
| [**CRI-O**](https://cri-o.io/) | Minimal runtime dedicated to Kubernetes [CRI](https://kubernetes.io/docs/concepts/containers/cri/). No Docker API and no `build`. |
| **containerd alone** | Can be used without `dockerd`, for example through `nerdctl`. |

## Summary

- `dockerd` is the **server**. It owns images, networking, volumes, and the API.
- "Docker daemon not running" means the server is not running. On a Mac, that usually means the VM is not running.
- It is **not** the thing that runs containers at the lowest level. It **delegates** to [containerd](./layer3_containerd.md), the next layer.
- Kubernetes skips this entire layer and talks directly to containerd.

## References

- Docker architecture, where dockerd owns images, networking, and volumes: <https://docs.docker.com/get-started/docker-overview/#docker-architecture>
- `dockerd` reference, `--containerd` gRPC address, daemon.json configuration, and Linux/Windows dependencies: <https://docs.docker.com/reference/cli/dockerd/>
- daemon.json configuration, including storage-driver and log-driver: <https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file>
- Historical context for dockershim removal and why Docker did not implement CRI: <https://kubernetes.io/blog/2022/05/03/dockershim-historical-context/>
- Dockershim removal FAQ: <https://kubernetes.io/blog/2022/02/17/dockershim-faq/>
- Kubernetes Container Runtime Interface (CRI) spec: <https://kubernetes.io/docs/concepts/containers/cri/>
- Kubernetes container runtime setup, including containerd and CRI-O: <https://kubernetes.io/docs/setup/production-environment/container-runtimes/>
- Podman, daemonless OCI container engine: <https://docs.podman.io/en/stable/markdown/podman.1.html>
