+++
title = "Layer 4 — runc (Low-level OCI Runtime)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "runc", "oci", "layers"]
+++

> Part 4 of the Docker layers series. From top to bottom, the stack is:
> 1. docker CLI → 2. dockerd → 3. containerd → **4. runc** →
> 5. Linux kernel. See the [overview article](./component_and_layers.md).

## Identity

`runc` is the **low-level runtime**: the program that *actually creates containers*.
Everything above it, including the CLI, dockerd, and containerd, is orchestration.
`runc` is the point where a container stops being an abstract idea and becomes a real,
isolated process on top of the [Linux kernel](./component_and_layers.md).

It is small, does one job, and **exits immediately** after starting the container. It is
the [reference implementation](https://github.com/opencontainers/runtime-spec/blob/main/implementations.md)
of the [**OCI Runtime Spec**](https://github.com/opencontainers/runtime-spec).

## What "creating a container" actually means

A container is not a virtual machine. It is an ordinary Linux process that has been
**lied to about the surrounding world**. `runc` tells that lie using three Linux kernel
features:

- [**namespaces**](https://man7.org/linux/man-pages/man7/namespaces.7.html): isolate
  *what the process can see*. The [OCI spec defines eight namespace types](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#namespaces):
  PID (its own PID 1), mount (its own filesystem view), network (its own interfaces),
  user, hostname (UTS), IPC, cgroup (its own cgroup view), and time (its own clock).
- [**cgroups**](https://man7.org/linux/man-pages/man7/cgroups.7.html), or control groups:
  [limit](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#control-groups)
  *what and how much the process can use*: CPU, memory, PID count, and I/O. This is how
  `--memory=512m` is enforced.
- **root filesystem, or rootfs**: `runc` uses `pivot_root` into the filesystem that
  containerd prepared from image layers, so the process sees the image's files as `/`
  instead of the host's files.

It also applies security layers:
[**capabilities**](https://github.com/opencontainers/runtime-spec/blob/main/config.md)
for dropping root privileges,
[**seccomp**](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#seccomp)
for syscall filtering, and optionally AppArmor/SELinux. Then it `exec`s the container's
entrypoint as PID 1 inside that constructed world, and exits.

## Why it almost never appears in `ps`

`runc` is **not long-lived**. It runs only while the container is being *created*, hands
off to the [shim](./layer3_containerd.md), and exits. So:

```bash
$ colima ssh -- sh -c 'ps -e -o comm | grep -E "dockerd|containerd|runc"'
dockerd
containerd
containerd-shim-runc-v2     # long-lived parent
# runc is usually not here — it already exited
```

To catch `runc` in the process list, you have to look while `docker run` is starting
something slow. After that, the container process is reparented to the shim and `runc`
disappears. This is intentional. See the shim explanation in [Layer 3](./layer3_containerd.md).

## OCI Runtime Spec: why runc can be replaced

`runc` is only the *reference* implementation of the **Open Container Initiative (OCI)
Runtime Specification**. The spec defines a standard
["bundle"](https://github.com/opencontainers/runtime-spec/blob/main/bundle.md), meaning
rootfs + `config.json`, and the
[commands](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md) that a
runtime must support, such as `create`, `start`, `kill`, and `delete`.

Because [containerd](./layer3_containerd.md) speaks through this spec, any OCI-compliant
runtime can be swapped in:

| Low-level runtime | What changes |
|---|---|
| [**runc**](https://github.com/opencontainers/runc) | Default. Standard Linux namespaces + cgroups. Written in Go. |
| [**crun**](https://github.com/containers/crun) | Same model, written in C: faster startup and lower memory use. |
| [**gVisor (`runsc`)**](https://gvisor.dev/docs/architecture_guide/security/) | Puts a **userspace kernel** in front of the real kernel: stronger isolation with some overhead. |
| [**Kata Containers**](https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md) | Runs each container inside a **lightweight micro VM**: VM-grade isolation with container ergonomics. |
| [**youki**](https://github.com/youki-dev/youki) | runc-compatible runtime written in Rust. |

Because they implement the same OCI interface, changing the runtime, for example for
stronger isolation, does **not** require touching the CLI, dockerd, or containerd. You
only change the runtime-class configuration.

## Seeing the spec consumed by runc

You can inspect runc directly inside the VM:

```bash
$ colima ssh -- sh -c 'command -v runc; runc --version'
/usr/bin/runc
runc version 1.x.x
spec: 1.x.x          # <- OCI Runtime Spec version it implements

# runc consumes a "bundle": rootfs directory + config.json
# config.json describes namespaces, cgroups, mounts, capabilities, and the process
```

That `config.json` is what containerd creates and passes to `runc`. `runc` reads it and
creates exactly the container described there.

## Full handoff flow

```text
docker run nginx
  └─ dockerd                    [Layer 2] image interpretation, network/volume setup
       └─ containerd            [Layer 3] image → unpack rootfs, create "task"
            └─ containerd-shim-runc-v2    (resident process)
                 └─ runc        [Layer 4] namespace/cgroup, pivot_root, exec → exit
                      └─ nginx  [       ] PID 1 inside the container
                           └─   [Layer 5] Linux kernel
```

## Summary

- `runc` is where a container becomes a **real isolated Linux process**: namespaces for what it sees, cgroups for what it uses, and rootfs for its filesystem.
- It is **short-lived**: it creates the container, hands off to the shim, and exits, so it usually does not appear in `ps`.
- It implements the **OCI Runtime Spec**, so it is **replaceable** without changing upper layers: crun, gVisor, Kata, youki.
- When using `runc`, the **Linux kernel** underneath it, Layer 5, is the non-swappable foundation. Its namespaces and cgroups make all of this possible. gVisor can abstract that foundation with a userspace kernel, and Kata can abstract it with a guest kernel.

## References

- runc, OCI Runtime Spec reference implementation with seccomp, AppArmor, and SELinux support: <https://github.com/opencontainers/runc>
- OCI Runtime Spec implementation list, where runc is the reference: <https://github.com/opencontainers/runtime-spec/blob/main/implementations.md>
- OCI Runtime Spec, eight namespace types: PID, mount, network, user, UTS, IPC, cgroup, time: <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#namespaces>
- OCI Runtime Spec, cgroup resource limits for CPU, memory, PIDs, and Block IO: <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#control-groups>
- OCI Runtime Spec, capabilities definition: <https://github.com/opencontainers/runtime-spec/blob/main/config.md>
- OCI Runtime Spec, seccomp filter configuration: <https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#seccomp>
- OCI Runtime Spec, bundle format with rootfs + config.json: <https://github.com/opencontainers/runtime-spec/blob/main/bundle.md>
- OCI Runtime Spec, lifecycle commands create, start, kill, delete: <https://github.com/opencontainers/runtime-spec/blob/main/runtime.md>
- Linux namespaces(7) man page: <https://man7.org/linux/man-pages/man7/namespaces.7.html>
- cgroups(7) man page: <https://man7.org/linux/man-pages/man7/cgroups.7.html>
- crun, written in C with 49.4% faster startup than runc: <https://github.com/containers/crun>
- gVisor security architecture, syscall interception through the userspace kernel Sentry: <https://gvisor.dev/docs/architecture_guide/security/>
- Kata Containers architecture, lightweight VM-based container isolation: <https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md>
- youki, OCI runtime written in Rust: <https://github.com/youki-dev/youki>
