+++
title = "Docker Components and Layers: docker, dockerd, containerd, runc, Colima, Lima"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "colima", "lima", "containerd"]
+++

This article organizes what actually happens when I run `docker ps` on a Mac using
Colima, and how docker / Docker Desktop / Colima / Lima / dockerd / containerd / runc fit
together as layers.

## Series index

These articles cover each layer in more detail. Layer 5, the Linux kernel, is covered only
in this overview.

1. [Layer 1 — docker CLI (Client)](./layer1_docker_cli.md)
2. [Layer 2 — dockerd (Daemon / API Server)](./layer2_dockerd.md)
3. [Layer 3 — containerd (Container Runtime)](./layer3_containerd.md)
4. [Layer 4 — runc (Low-level OCI Runtime)](./layer4_runc.md)
5. Layer 5 — Linux kernel, covered in this article

## The bug that started this

When I ran `docker ps`, I got ``Command `docker` not found``. The cause was **not**
Docker itself:

- Colima was installed and running correctly (`runtime: docker`).
- The `docker` CLI was also installed through Homebrew
  (`/opt/homebrew/Cellar/docker/<ver>/bin/docker`).
- But it was **not symlinked into** `/opt/homebrew/bin/`, so it was missing from `$PATH`.

The fix:

```bash
brew link --overwrite docker
```

`brew link` creates symlinks from the files installed in a formula's "Cellar" into
Homebrew's `bin/`, which is on `$PATH`. `--overwrite` replaces any existing destination
file. In this case, that was likely leftover Docker Desktop state, because `~/.docker`
still existed.

## Core idea

Docker/OCI containers are **Linux** technology. They need the Linux kernel. A Mac does not
have a Linux kernel, so on macOS, everything below the CLI exists to "create a hidden
Linux VM and let the Mac talk to it."

## The full stack, top to bottom

```text
┌─────────────────────────────────────────────────────────────┐
│  1. docker CLI            "client" — what I type              │  ← me
│     (docker ps, run...)                                       │
│            │ speaks Docker API (HTTP) through a socket         │
│            ▼                                                   │
├─────────────────────────────────────────────────────────────┤
│  2. dockerd               "server/daemon" — receives API       │
│     (Docker daemon)        requests, manages images/networking │
├─────────────────────────────────────────────────────────────┤
│  3. containerd            container runtime — pulls images,    │
│                            manages container lifecycle         │
├─────────────────────────────────────────────────────────────┤
│  4. runc                  low-level OCI runtime — actually     │
│                            creates containers with kernel      │
│                            features (namespace, cgroup)        │
├─────────────────────────────────────────────────────────────┤
│  5. Linux kernel          what containers truly need           │
│     (inside a VM on macOS)                                     │
└─────────────────────────────────────────────────────────────┘
```

Docker Engine itself is "server + api + client" (Layers 1 and 2). Docker was originally
**monolithic**, with Layers 2 through 4 inside one program. As Docker modularized its
architecture, `containerd` was split out (Docker 1.11, 2016). Kubernetes' CRI, or
Container Runtime Interface, strengthened that separation further, allowing systems such
as Kubernetes to use `containerd` directly without the full Docker engine.

### What each layer owns

| # | Component | What it does |
|---|---|---|
| 1 | **docker CLI** | *Client*. Converts my command into a Docker API (HTTP) request. It does not run anything by itself. |
| 2 | **dockerd** | *Server/daemon*. Receives API requests and manages images, networking, and volumes. Delegates actual execution downward. |
| 3 | **containerd** | *Container runtime*. Pulls/stores images, tracks container state, and supervises lifecycle. |
| 4 | **runc** | *Low-level runtime*. Creates actual containers using Linux kernel features: namespaces and cgroups. Runs briefly during container start, then exits. |
| 5 | **Linux kernel** | The essence of containers: isolated processes on top of the Linux kernel. On macOS, it exists only inside a VM. |

## socket: the path between CLI and daemon

The docker CLI and daemon are separate programs. They communicate through a **Unix domain
socket**. It looks like a file, but it does not store data. It is a live connection point,
more like a door than a drawer.

```text
$ ls -l ~/.colima/default/docker.sock
srw------- 1 bds0900 staff 0 Jun 11 13:05 ...docker.sock
│          └─ size 0: nothing is stored. Data only passes through.
└─ leading 's' = socket, not a regular file '-' or directory 'd'
```

- `s` flag → it is definitely a socket. `stat` reports `type: Socket`, and `file` says `socket`.
- Size `0` → it is a passage, not storage.
- `srw-------` → in this Colima environment, only the owner can talk to the daemon. Standard Linux Docker usually uses `srw-rw----` so members of the `docker` group can also access it.

When I run `docker ps`, the CLI opens this socket, writes an HTTP request
(`GET /containers/json`), and reads the response. The socket file is on the **Mac**, but
the daemon it reaches runs inside the **Linux VM**. Colima bridges that boundary.

## Docker Desktop vs Colima vs Lima

These three are not new layers. They are three different ways to provide **Layers 2
through 5**, meaning the VM and the daemon stack inside it. The top `docker` CLI, Layer 1,
is the same for all three.

```text
        ┌──────────── docker CLI (Layer 1) ────────────┐
        │   talks to the daemon running underneath      │
        └───────────────────────────────────────────────┘
                            ▲  ▲  ▲
          ┌─────────────────┘  │  └──────────────────┐
 ┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
 │ Docker Desktop │  │     Colima       │  │   raw Lima       │
 │ own VM + full  │  │  wraps Lima ─────┼──┼──► provides a    │
 │ daemon + GUI   │  │  ("runtime proxy")│  │   Linux VM       │
 └────────────────┘  └──────────────────┘  └──────────────────┘
```

- **Lima** (*Li*nux *ma*chines) provides the Linux virtual machine layer. One of its goals is to promote containerd.
- **Colima** (*Co*ntainers on *Lima*) wraps **Lima** as a "container runtime proxy." It asks Lima to create a Linux VM, install the daemon, and expose the socket.
- **Docker Desktop** bundles everything corresponding to Layers 2 through 5 itself, plus GUI, settings UI, auto-updates, Compose, and an optional Kubernetes cluster.

| | **Docker Desktop** | **Colima** | **Lima (raw)** |
|---|---|---|---|
| VM provider | Own VM | Through **Lima** | **Itself** |
| dockerd/containerd provided | Bundled | Set up inside the VM | Manual setup |
| Orientation | GUI + all-in-one | Container-first, CLI | General-purpose Linux VM |
| Relationship | Independent | **Colima → Lima → VM** | Foundation |

The main reasons to move from Docker Desktop to Colima are **licensing** (larger
companies need paid Docker Desktop subscriptions, while Colima is free OSS) and a
**lighter CLI-centered workflow** with no always-running GUI app.

## Products that fill each slot

Each layer is not a fixed product, but a swappable slot:

| # | Layer (role) | Standard tool | Other products |
|---|---|---|---|
| 1 | Client (CLI) | `docker` | `nerdctl`, `podman`, `crictl` |
| 2 | Daemon / API server | `dockerd` | Podman, daemonless |
| 3 | Runtime, high-level | `containerd` | CRI-O |
| 4 | Runtime, low-level OCI | `runc` | `crun`, gVisor (`runsc`), Kata, `youki` |
| 5 | Linux kernel | Linux | Provided inside the VM |
| — | VM engine | **Docker Desktop** | Lima, Rancher Desktop, Podman Machine, Minikube |
| — | Container wrapper | **Docker Desktop** | Colima, Rancher Desktop, OrbStack |
| — | Hypervisor on macOS | Apple Virtualization.framework | Hypervisor.framework, QEMU |

## Mapping to this Mac

```text
 Layer 1  client              →  docker            (Homebrew, /opt/homebrew/bin/docker)
 ─ socket ─                  →  docker.sock       (~/.colima/default/docker.sock)
 Layer 2  daemon/API          →  dockerd           (inside VM)
 Layer 3  runtime (high-level) →  containerd        (inside VM)
 Layer 4  runtime (low-level)  →  runc              (inside VM)
 Layer 5  Linux kernel         →  Linux (aarch64)   (inside VM)
 ─────────────────────────────────────────────────────────────────
 VM wrapper                   →  Colima            ("runtime proxy")
 VM engine                    →  Lima              (boots the VM)
 Hypervisor                   →  Apple Virtualization.framework
```

`colima status` shows that the VM runs on `aarch64` using *macOS
Virtualization.Framework*, and that the docker socket is at
`~/.colima/default/docker.sock`.

To see Layers 2 through 4 running inside the VM:

```bash
colima ssh -- sh -c 'ps -e -o pid,comm | grep -E "dockerd|containerd|runc"'
# dockerd and containerd stay running.
# runc is usually not visible because it runs briefly during container start and exits.
# You can catch it only during docker run.
```

## One-paragraph summary

The vertical stack that actually runs containers is:
`docker CLI → dockerd → containerd → runc → Linux kernel`. On a Mac, that kernel exists
only inside a **Linux VM**. **Docker Desktop, Colima, and Lima are not steps in this
stack; they are competing ways to provide the stack.** Lima provides the VM. Colima uses
Lima to set up the daemon stack as a container-focused wrapper. Docker Desktop is an
all-in-one GUI app bundling its own VM and daemon stack. The top-level `docker` CLI does
not care which one you chose. It simply opens a socket and speaks Docker API to whichever
daemon is listening.

## References

- emptyfridge.dev — Docker component architecture: <https://emptyfridge.dev/docker/docker/>
