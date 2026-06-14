# Docker Deep Dive — Nigel Poulton

**Book**: *Docker Deep Dive* (May 2025 edition) | **Author**: Nigel Poulton | **~280 pages** | **17 chapters**

**Buy the book**: [leanpub.com/dockerdeepdive](https://leanpub.com/dockerdeepdive)

Structured chapter summaries extracted for agent reference. Load individual chapters on demand.

## How to Use

- Ask about a **topic** (e.g. `overlay networking`, `multi-stage builds`, `Docker Model Runner`) — read the matching chapter file first
- Ask for a **chapter number** (e.g. `ch08`) — load that file directly
- Browse the **chapter index** below for full coverage

## Chapter Index

| # | File | Title | Key Topics |
|---|------|-------|------------|
| 0 | [ch00-about-the-book](chapters/ch00-about-the-book.md) | About the book | Two-part structure, chapter roadmap, update cadence |
| 1 | [ch01-containers-30000-feet](chapters/ch01-containers-30000-feet.md) | Containers from 30,000 feet | VM → container evolution, Linux/Windows/Mac, Wasm, AI, K8s |
| 2 | [ch02-standards-and-projects](chapters/ch02-standards-and-projects.md) | Standards and projects | Docker platform vs Inc., OCI, CNCF, Moby |
| 3 | [ch03-getting-docker](chapters/ch03-getting-docker.md) | Getting Docker | Docker Desktop, Multipass, Linux install |
| 4 | [ch04-the-big-picture](chapters/ch04-the-big-picture.md) | The big picture | Ops vs Dev perspectives, first pull/build/run |
| 5 | [ch05-docker-engine](chapters/ch05-docker-engine.md) | The Docker Engine | dockerd, containerd, runc, shim, OCI |
| 6 | [ch06-working-with-images](chapters/ch06-working-with-images.md) | Working with Images | Layers, registries, digests, multi-arch, Scout |
| 7 | [ch07-working-with-containers](chapters/ch07-working-with-containers.md) | Working with containers | Lifecycle, exec, debug, restart policies |
| 8 | [ch08-containerizing-an-app](chapters/ch08-containerizing-an-app.md) | Containerizing an app | Dockerfile, multi-stage, BuildKit, buildx |
| 9 | [ch09-compose](chapters/ch09-compose.md) | Multi-container apps with Compose | compose.yaml, service discovery, deploy |
| 10 | [ch10-docker-and-ai](chapters/ch10-docker-and-ai.md) | Docker and AI | Docker Model Runner, local LLMs |
| 11 | [ch11-docker-and-wasm](chapters/ch11-docker-and-wasm.md) | Docker and Wasm | Spin runtime, wasi/wasm builds |
| 12 | [ch12-docker-swarm](chapters/ch12-docker-swarm.md) | Docker Swarm | init/join, stacks, overlay deploy |
| 13 | [ch13-docker-networking](chapters/ch13-docker-networking.md) | Docker Networking | CNM, bridge, port mappings, MACVLAN |
| 14 | [ch14-overlay-networking](chapters/ch14-overlay-networking.md) | Overlay networking | VXLAN, VTEP, encrypted overlays |
| 15 | [ch15-volumes](chapters/ch15-volumes.md) | Volumes and persistent data | Volumes vs ephemeral, bind mounts |
| 16 | [ch16-docker-security](chapters/ch16-docker-security.md) | Docker security | Namespaces, cgroups, Scout, DCT, secrets |

## Topic Index

- **BuildKit / buildx** → ch08
- **Capabilities / seccomp** → ch16
- **CNCF / OCI** → ch02, ch05
- **Compose** → ch09
- **containerd / runc / shim** → ch05
- **Docker Desktop** → ch03
- **Docker Debug** → ch07
- **Docker Model Runner** → ch10
- **Docker Scout** → ch06, ch16
- **Image layers / digests** → ch06
- **Multi-stage builds** → ch08
- **Overlay / VXLAN** → ch14
- **Port mappings** → ch13
- **Restart policies** → ch07
- **Swarm** → ch12, ch14
- **Volumes** → ch15
- **Wasm** → ch11

## Supporting Files

- [glossary.md](glossary.md) — key terms with chapter references
- [patterns.md](patterns.md) — techniques and workflows
- [cheatsheet.md](cheatsheet.md) — decision rules and quick reference

## Scope

Covers book content only. For Coolify/Hetzner deployment patterns, combine with project docs in `AGENTS.md` and related references.
