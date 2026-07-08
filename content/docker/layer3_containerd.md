+++
title = "Layer 3 — containerd (Container Runtime)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "containerd", "runtime", "layers"]
+++

> Part 3 of the Docker layers series. From top to bottom, the stack is:
> 1. docker CLI → 2. dockerd → **3. containerd** → 4. runc →
> 5. Linux kernel. See the [overview article](./component_and_layers.md).

## Identity

`containerd` is a **container runtime**: the daemon that sits between
[dockerd](./layer2_dockerd.md) above and [runc](./layer4_runc.md) below. It handles the
hard parts of the container lifecycle: pulling and storing image content, unpacking it
into a filesystem, and supervising containers from start to stop.

It is a CNCF [**graduated project**](https://www.cncf.io/announcements/2019/02/28/cncf-announces-containerd-graduation/)
and an industry standard. The same `containerd` powers Docker, most Kubernetes clusters,
and tools such as `nerdctl`. That is exactly why it was split out of Docker.

## Why it exists: breaking up the monolith

Docker was originally monolithic. As Docker
[modularized its architecture](https://www.docker.com/blog/docker-engine-1-11-runc/),
the runtime became an independent project (Docker 1.11, 2016). Later, Kubernetes'
[**CRI, or Container Runtime Interface**](https://kubernetes.io/docs/concepts/containers/cri/),
reinforced the split so other systems could use the runtime **without** the whole Docker
engine:

```text
        dockerd   ── needs ──►  containerd  ◄── needs ──  Kubernetes (kubelet)
       (Docker)                  (shared)                 (through CRI)
```

So `containerd` serves two very different owners through two APIs:

- **dockerd** talks through containerd's [**native gRPC API**](https://github.com/containerd/containerd/blob/main/RELEASES.md).
- **Kubernetes** talks through [**CRI**](https://github.com/containerd/containerd/blob/main/docs/cri/architecture.md), the gRPC API defined by Kubernetes.

Because of this double role, dropping `dockerd` in Kubernetes ("dockershim removal") did
not change the actual runtime. Kubernetes simply talks directly to the same `containerd`.

## What it owns

`containerd` owns runtime concerns: a layer more concrete than `dockerd`, but more
abstract than `runc`.

- **Image management**: pulls images, verifies digests, stores **content** in a content-addressable store, and manages snapshots/layers.
- **Snapshots**: prepares container root filesystems from image layers through a [*snapshotter*](https://github.com/containerd/containerd/blob/main/docs/snapshotters/README.md), such as `overlayfs`.
- **Container lifecycle**: create, start, pause, stop, delete, and **supervise** running processes so they survive even if `dockerd` restarts.
- **Task**: the actual running instance of a container.
- **Shim**: a small [`containerd-shim-runc-v2`](https://github.com/containerd/containerd/blob/main/docs/runtime-v2.md) process attached to each container. It separates container I/O and exit status from the main containerd daemon.

It does not directly create namespaces or cgroups. That final step is delegated to
[runc](./layer4_runc.md).

## shim: why containerd can restart safely

The key design point: `containerd` does not remain the direct parent of every container.
Instead, it starts a **shim** for each container:

```text
containerd
   └─ containerd-shim-runc-v2   ← lives for the container's lifetime
         └─ runc  (runs briefly to create the container, then exits)
               └─ my container process (PID 1 inside the container)
```

Because the long-lived parent is the shim, not containerd:

- **Running containers do not die when `containerd` or `dockerd` restarts or upgrades**. The shim keeps them alive and reconnects.
- The shim captures the container's **stdout/stderr** and reports the **exit code** back to containerd.

This is the structural reason `runc` rarely appears in `ps`: after it finishes its job and
exits, the shim takes over.

## Seeing it on this Mac

`containerd` runs inside the Colima VM:

```bash
$ colima ssh -- sh -c 'command -v containerd; ctr --version'
/usr/bin/containerd
ctr github.com/containerd/containerd v1.7.x

# `ctr` is containerd's low-level debug CLI, separate from docker:
$ colima ssh -- sudo ctr namespaces ls
NAME    LABELS
default
moby            # <- namespace used by dockerd inside containerd
```

`moby` is the [containerd namespace](https://github.com/containerd/containerd/blob/main/docs/namespaces.md)
used by Docker. [Moby](https://github.com/moby/moby) is the name of Docker's upstream
open-source project. Seeing this namespace shows that `dockerd` is controlling
`containerd` underneath.

## containerd and surrounding layers

| Concern | Owner |
|---|---|
| `docker build`, networking, volumes, Docker API | **dockerd** (Layer 2) |
| Image pull/storage, snapshots, lifecycle, shim | **containerd** (Layer 3) |
| Namespace/cgroup creation, process `exec` | **runc** (Layer 4) |

## Alternatives that fill this slot

- [**CRI-O**](https://github.com/cri-o/cri-o): a runtime built *only* for Kubernetes CRI. It is lighter than containerd for that single purpose and has no Docker API.
- **containerd with another low-level runtime**: through runtime classes, containerd can choose `runc`, `crun`, gVisor (`runsc`), or Kata. See [Layer 4](./layer4_runc.md).

## Summary

- `containerd` is the **standard runtime** for image content, snapshots, and container lifecycle.
- It was **split out of Docker** so Kubernetes and other systems could use it directly through CRI and bypass `dockerd`.
- The **shim** design lets containers keep running even when daemons restart.
- The final low-level container creation is still delegated to [runc](./layer4_runc.md), the next layer.

## References

- CNCF containerd graduation announcement (2019): <https://www.cncf.io/announcements/2019/02/28/cncf-announces-containerd-graduation/>
- Docker 1.11, the first release to move to containerd/runc: <https://www.docker.com/blog/docker-engine-1-11-runc/>
- Docker, "Introducing containerd" and the modularization background: <https://www.docker.com/blog/introducing-containerd/>
- containerd CRI plugin architecture for Kubernetes integration: <https://github.com/containerd/containerd/blob/main/docs/cri/architecture.md>
- containerd native gRPC API, "the primary product of containerd": <https://github.com/containerd/containerd/blob/main/RELEASES.md>
- Snapshotter, preparing rootfs from image layers: <https://github.com/containerd/containerd/blob/main/docs/snapshotters/README.md>
- Content flow, from image pull to rootfs: <https://github.com/containerd/containerd/blob/main/docs/content-flow.md>
- Runtime v2 shim, container survival across daemon restarts: <https://github.com/containerd/containerd/blob/main/docs/runtime-v2.md>
- containerd namespaces, including `moby` multitenancy: <https://github.com/containerd/containerd/blob/main/docs/namespaces.md>
- Moby, Docker's upstream open-source project: <https://github.com/moby/moby>
- CRI-O, lightweight runtime dedicated to Kubernetes CRI: <https://github.com/cri-o/cri-o>
