+++
title = "Layer 1 — docker CLI (Client)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["docker", "cli", "layers"]
+++

> Part 1 of the Docker layers series. From top to bottom, the stack is:
> **1. docker CLI (client)** → 2. dockerd (daemon) → 3. containerd → 4. runc →
> 5. Linux kernel. See the [overview article](./component_and_layers.md).

## Identity

The `docker` CLI is the program where I type commands: `docker ps`, `docker run`,
`docker build`. It is **only a client**. It does not directly run, build, or store
containers. Its only job is to turn my command into a
[Docker API](https://docs.docker.com/get-started/docker-overview/#docker-architecture)
request, send it to the daemon, receive the response, and print it nicely on the screen.

A useful analogy: the CLI is a **remote control**. When you press a button, it sends a
signal, and the TV does the real work. `docker` is the remote control, and `dockerd` is
the TV.

## Where it is

```bash
$ which docker
/opt/homebrew/bin/docker          # Homebrew symlink...
$ ls -l /opt/homebrew/bin/docker
... -> ../Cellar/docker/29.5.3/bin/docker   # ...pointing to the real binary
```

On this Mac, the CLI came from **Homebrew** (`brew install docker`), not Docker Desktop.
That distinction was the cause of the earlier "command not found" bug: older Docker
Desktop installs bundled their own `docker` CLI, and when Docker Desktop was removed,
that bundled CLI disappeared too. The standalone Homebrew CLI existed, but it was not
linked into `$PATH` until I ran `brew link --overwrite docker`.

**Key point:** the CLI is independent of the engine. The same `docker` binary can connect
to Docker Desktop, Colima, or a remote server, as long as the target speaks the Docker API.

## What happens when a command is typed

`docker ps` does not run "list containers" code locally. It is a
[`GET /containers/json`](https://docs.docker.com/reference/api/engine/version/v1.47/#tag/Container/operation/ContainerList)
API call:

```text
docker ps
  └─ CLI creates an HTTP request:    GET /v1.43/containers/json
  └─ opens a socket:                 unix:///Users/you/.colima/default/docker.sock
  └─ writes the request and reads the JSON response
  └─ formats the JSON into the table we see
```

You can prove that the CLI is just an HTTP client by talking to the daemon manually:

```bash
# Same result as `docker ps`, without using the docker CLI
curl --unix-socket ~/.colima/default/docker.sock http://localhost/v1.43/containers/json
```

The same container data that `docker ps` turns into a table comes back.

## How the CLI chooses which daemon to talk to

The CLI chooses *which* daemon to talk to using this priority order:

1. **`-H` / `--host` or `--context` flag** — explicit, for example `docker -H tcp://1.2.3.4:2375 ps`
2. **`DOCKER_CONTEXT` environment variable** — when set, it overrides both `DOCKER_HOST` and the active context
3. **Active context** — `docker context ls` / `docker context use`
4. **`DOCKER_HOST` environment variable** — for example `unix:///path/to/docker.sock`
5. **Default** — `/var/run/docker.sock`

In a Colima environment, Colima registers a **context** pointing to
`~/.colima/default/docker.sock`, or sets `DOCKER_HOST`. That is why the CLI reaches the
daemon inside the VM even without flags:

```bash
$ docker context ls
NAME       DESCRIPTION                  DOCKER ENDPOINT
colima *   colima                       unix:///Users/you/.colima/default/docker.sock
default    Current DOCKER_HOST based    unix:///var/run/docker.sock
```

The `*` marks the active context. Changing the context is how the same CLI can connect to
another engine, such as Docker Desktop, without reinstalling anything.

## The Docker API spoken by the CLI

The CLI and daemon communicate through the
[**Docker Engine API**](https://docs.docker.com/reference/api/engine/): a versioned REST
API spoken over HTTP.

- **Versioned**: requests include a version (`/v1.43/...`). Even a newer CLI can
  [negotiate down](https://docs.docker.com/reference/api/engine/) and talk to an older daemon.
- **Transport-independent**: the same API runs over a Unix socket locally or a TCP socket remotely. The CLI does not care which transport is used.
- **Not magic**: everything the CLI does over the socket can also be done with `curl`. The CLI is a convenience formatter over the API.

## Other tools that fill this slot

Because the daemon exposes a standard API, clients can be swapped:

| Client | Notes |
|---|---|
| `docker` | Standard CLI. |
| [`nerdctl`](https://github.com/containerd/nerdctl) | Docker-compatible CLI that talks directly to **containerd**. |
| `podman` | Daemonless. Its CLI is mostly Docker-compatible (`alias docker=podman`). |
| [`crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) | CRI-focused CLI used for Kubernetes debugging. |
| Docker SDK | Go/Python and other libraries that call the same API from code. |

## Summary

- The CLI is a **thin client**: it sends API requests and formats responses.
- It is **separate from the engine**. One CLI can target several daemons with `--host`, `DOCKER_HOST`, or contexts.
- The earlier "command not found" bug was not an engine problem. It was a **client packaging/PATH** problem.
- One layer below: [dockerd, the daemon](./layer2_dockerd.md), which actually receives these API requests.

## References

- Docker architecture, client-server structure and CLI-to-daemon API flow: <https://docs.docker.com/get-started/docker-overview/#docker-architecture>
- `docker` CLI reference, `--host`, `DOCKER_CONTEXT`, and `DOCKER_HOST` priority: <https://docs.docker.com/reference/cli/docker/>
- Docker Engine API version negotiation, "Versioned API and SDK": <https://docs.docker.com/reference/api/engine/>
- `GET /containers/json`, the `docker ps` API reference: <https://docs.docker.com/reference/api/engine/version/v1.47/#tag/Container/operation/ContainerList>
- Example of curling over a Unix socket directly: <https://docs.docker.com/reference/api/engine/sdk/examples/>
- nerdctl, Docker-compatible CLI for containerd: <https://github.com/containerd/nerdctl>
- crictl, CRI CLI for Kubernetes node debugging: <https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/>
