# Glossary — Docker Deep Dive

**BuildKit** — OCI-compliant image builder used by modern Docker; supports cache mounts, secrets, multi-stage (Ch 5, 8)

**Buildx** — CLI plugin for advanced builds including multi-platform and remote builders (Ch 8)

**Capabilities** — Fine-grained Linux privilege drops; prefer dropping over running privileged (Ch 16)

**CNCF** — Cloud Native Computing Foundation; hosts Kubernetes, containerd, Prometheus; sandbox → incubating → graduated (Ch 2)

**CNM (Container Network Model)** — Sandboxes, endpoints, networks; implemented by libnetwork (Ch 13)

**containerd** — CNCF graduated high-level runtime; lifecycle, images, networks, volumes (Ch 5)

**Containerized app** — Application running as a container (Ch 1)

**Control groups (cgroups)** — Kernel resource limits for CPU, memory, I/O (Ch 1, 16)

**Daemonless containers** — Containers survive dockerd restart because shim owns the process (Ch 5)

**Docker Content Trust (DCT)** — Notary-based image signing and verification (Ch 16)

**Docker Debug** — Sidecar debugging for slim/production images without adding tools to the image (Ch 7)

**Docker Desktop** — Packaged Docker with UI, extensions, Compose, optional K8s; Mac runs engine in hidden Linux VM (Ch 3)

**Docker Engine** — Server-side components: API, builder, containerd, shims, runc (Ch 5)

**Docker Hub** — OCI distribution-spec compliant registry (Ch 2, 6)

**Docker Model Runner (DMR)** — Runs local LLMs outside containers for direct GPU/hardware access (Ch 10)

**Docker Scout** — Vulnerability scanning and SBOM analysis for images (Ch 6, 16)

**Embedded DNS** — Network-scoped name resolution on user-defined bridge/overlay networks; not on default bridge (Ch 13)

**Image digest** — Immutable SHA256 content address; pin with `@sha256:...` (Ch 6)

**Image layer** — Read-only filesystem diff; shared across images; copy-on-write at container runtime (Ch 6, 7)

**Ingress network** — Hidden Swarm overlay enabling routing mesh for published ports (Ch 13)

**Kernel namespaces** — PID, NET, MNT, UTS, IPC, USER isolation (Ch 1, 16)

**libcontainer** — Platform-agnostic kernel interface replacing LXC in Docker Engine (Ch 5)

**Moby Project** — Community toolkit for composing container platforms (Ch 2)

**Multi-stage build** — Separate build and runtime stages; smaller production images (Ch 8)

**OCI** — Open Container Initiative; image-spec, runtime-spec, distribution-spec (Ch 2, 5)

**Overlay network** — Multi-host network via VXLAN encapsulation (Ch 14)

**Port mapping** — `-p host:container` for external access; dev-friendly, not scale pattern (Ch 13)

**Restart policy** — `no`, `on-failure`, `always`, `unless-stopped` for self-healing (Ch 7)

**runc** — OCI runtime-spec reference implementation; creates container then exits (Ch 5)

**seccomp** — Syscall filtering profile for containers (Ch 16)

**Shim (containerd-shim-runc-v2)** — Parent process after runc exits; keeps I/O and reports status (Ch 5)

**Swarm mode** — Built-in Docker orchestration; managers + workers, stacks, secrets (Ch 12, 16)

**TLDR section** — Poulton's chapter opener summarizing essentials before deep dive (throughout)

**Volume** — Docker-managed persistent storage decoupled from container lifecycle (Ch 15)

**VXLAN / VTEP** — Overlay encapsulation; virtual tunnel endpoints on each host (Ch 14)

**Wasm (WebAssembly)** — Binary instruction set; Docker supports via Spin runtime and wasi/wasm platform (Ch 11)
