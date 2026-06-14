# Chapter 5: The Docker Engine

## Core Idea

The Docker Engine is jargon for the server-side components that build, share, and run containers—analogous to VMware ESXi for VMs. It is modular, OCI-compliant, and composed of many small specialized tools (API, image builder, high-level runtime, low-level runtime, shims) rather than a monolithic daemon. Understanding this stack is optional for day-to-day use but essential for mastering Docker.

## Frameworks Introduced

- **Monolith → modular refactor**: Docker stripped container creation, image management, and runtime work out of `dockerd` into composable tools (`containerd`, `runc`, BuildKit) so platform builders can reuse them and running containers survive daemon restarts.
- **High-level vs low-level runtime split**: `containerd` manages lifecycle events (start, stop, delete, images, networks, volumes); `runc` interfaces with the kernel to actually construct containers from OCI bundles.
- **OCI layer terminology**: runc operates at the **OCI layer** as the reference implementation of the **runtime-spec**; BuildKit builds **image-spec** images; Docker Hub is an **distribution-spec** registry.
- **Daemonless containers**: Decoupling container processes from `dockerd` lets you stop, restart, or upgrade the daemon without killing running containers.

## Key Concepts

- Original Engine: monolithic **Docker daemon** + **LXC** (Linux-only, slow to evolve).
- **libcontainer** replaced LXC as a platform-agnostic kernel interface (namespaces, cgroups).
- **Open Container Initiative (OCI)**: image-spec, runtime-spec (v1.0 July 2017), distribution-spec—all Docker since 2016 implements these.
- **runc**: reference OCI runtime-spec implementation; lightweight CLI wrapper for libcontainer; exits after container starts; paired with containerd everywhere (Docker, Kubernetes, Firecracker, Fargate).
- **containerd**: CNCF graduated project; high-level runtime; modular (Kubernetes can take only needed pieces); grew to include image/network/volume management; Docker Desktop 4.27.0+ uses containerd for image management instead of the daemon.
- **Shim** (`containerd-shim-runc-v2`): sits between containerd and OCI layer; parent of running container after runc exits; keeps STDIN/STDOUT open; reports status; enables pluggable low-level runtimes.
- **Communication**: Docker client → API → `dockerd` → gRPC CRUD API → `containerd` → OCI bundle → `runc` → kernel.
- **Linux binaries**: `/usr/bin/dockerd`, `/usr/bin/containerd`, `/usr/bin/containerd-shim-runc-v2`, `/usr/bin/runc`.
- **API sockets**: Linux `/var/run/docker.sock`; Windows `\pipe\docker_engine`.
- Daemon still serves the **Docker API** even after most functionality moved out.

## Mental Models

- **Car engine analogy**: Specialized parts (intake, pistons, exhaust) combine to drive; Engine parts (API, builder, runtimes, shims) combine to run containers.
- **Container birth sequence**: Client request → daemon interprets → containerd converts image to OCI bundle → runc builds namespaces/cgroups → container starts as runc child → runc exits → shim becomes parent.
- **Layer cake**: User-facing Docker CLI experience sits atop OCI-compliant plumbing you rarely touch directly.

## Anti-patterns

- **Treating `dockerd` as the container creator**: Container creation code lives in containerd/runc; the daemon only forwards requests.
- **Expecting to see shim/runc processes on Docker Desktop Mac**: Engine runs inside a VM—`ps` on the Mac host won't show them.
- **Ignoring OCI portability**: Assuming Docker-specific internals instead of standards (image-spec, runtime-spec) limits cross-runtime reasoning.

## Code Examples

Start a container (triggers full Engine stack):

```bash
$ docker run -d --name ctr1 nginx
```

Verify running:

```bash
$ docker ps
CONTAINER ID   IMAGE   COMMAND                  CREATED         STATUS         PORTS     NAMES
9cfb0c9aacb2   nginx   "/docker-entrypoint.…"   9 seconds ago   Up 9 seconds   80/tcp    ctr1
```

Clean up:

```bash
$ docker rm ctr1 -f
```

Inspect Engine processes on Linux (when containers are running):

```bash
$ ps aux | grep -E 'dockerd|containerd|runc|containerd-shim'
```

## Reference Tables

| Component | Role | Layer |
|-----------|------|-------|
| `dockerd` | Docker API server | Control plane |
| `containerd` | Lifecycle, images, networks, volumes | High-level runtime |
| `containerd-shim-runc-v2` | Parent process, I/O, status after runc exits | Shim |
| `runc` | Kernel interface, OCI bundle → container | Low-level runtime (OCI layer) |
| BuildKit | OCI image builds | Build |
| OCI image-spec | Image format standard | Standard |
| OCI runtime-spec | Container runtime standard | Standard |
| OCI distribution-spec | Registry distribution standard | Standard |

| Era | Architecture |
|-----|--------------|
| Early Docker | Monolithic daemon + LXC |
| libcontainer era | Monolithic daemon + libcontainer |
| Modern | Modular daemon + containerd + runc + shims + BuildKit |

## Worked Example

Run `docker run -d --name ctr1 nginx`, then trace: CLI converts to API POST → `dockerd` on `/var/run/docker.sock` passes to containerd via gRPC → containerd builds OCI bundle from nginx image layers → runc creates namespaces/cgroups and starts nginx as PID 1 → runc exits → shim remains parent → `docker ps` shows running container. Stop/restart `dockerd` on Linux—the container keeps running (daemonless). `docker rm ctr1 -f` tears down shim and container.

## Key Takeaways

1. Docker Engine = modular server-side stack, not a single binary.
2. OCI specs (image, runtime, distribution) underpin interoperability since 2016.
3. containerd (high-level) + runc (low-level OCI) is the universal pairing in Docker and Kubernetes.
4. Shims enable daemonless containers and swappable OCI runtimes.
5. Image management moved from daemon to containerd (Docker Desktop 4.27.0+).
6. Mastering Docker means knowing what happens between `docker run` and a running process.

## Connects To

- **Chapter 6**: Images are OCI bundles that containerd presents to runc.
- **Chapter 7**: `docker run` lifecycle maps directly to this Engine stack.
- **Chapter 8**: BuildKit (Engine build component) produces OCI images via `docker build`.
- **Chapter 2 (book)**: OCI, CNCF, Moby project context.
- **Coolify/Hetzner**: Production hosts run the same Engine stack; daemon upgrades shouldn't kill workloads when shims are in play.
