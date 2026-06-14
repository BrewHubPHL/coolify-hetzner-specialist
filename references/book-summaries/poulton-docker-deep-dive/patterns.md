# Patterns — Docker Deep Dive

## Ops Perspective Loop

**When to use**: First hands-on with any image on a host.

**How**:
1. `docker pull <image>`
2. `docker run -d --name <name> <image>`
3. `docker exec -it <name> <cmd>` to inspect
4. `docker stop <name>` → `docker rm <name>`

**Trade-offs**: Fast for exploration; no Dockerfile or persistence yet.

## Dev Perspective Loop

**When to use**: Containerizing application source code.

**How**:
1. Clone/write app source
2. Author `Dockerfile` (FROM, COPY, RUN, CMD/ENTRYPOINT)
3. `docker build -t <tag> .`
4. `docker run -p <host>:<container> <tag>`

**Trade-offs**: Reproducible builds; requires Dockerfile discipline.

## Multi-Stage Production Image

**When to use**: Compiled languages or Node/Python apps with large build toolchains.

**How**:
1. Stage 1 (`AS builder`): install deps, compile
2. Stage 2: `FROM` minimal base; `COPY --from=builder` artifacts only
3. Build with BuildKit enabled

**Trade-offs**: Smaller attack surface and faster deploys; more complex Dockerfile.

## User-Defined Bridge + DNS

**When to use**: Multi-container apps on a single host needing hostname resolution.

**How**:
1. `docker network create -d bridge <net>`
2. Run containers with `--network <net> --name <svc>`
3. Reach peers by name (not IP)

**Trade-offs**: Avoid default `bridge` (no DNS). Still single-host only.

## Compose Multi-Service Deploy

**When to use**: Local dev or small multi-service stacks.

**How**:
1. Define `compose.yaml` with services, networks, volumes
2. `docker compose up -d`
3. `docker compose ps` / `logs` / `down`

**Trade-offs**: Simple orchestration; Swarm/K8s needed for production scale.

## Pin Image by Digest

**When to use**: Production reproducibility and supply-chain control.

**How**: `docker pull nginx@sha256:<digest>` or reference digest in deploy manifests.

**Trade-offs**: Immutable but must update digest for patches.

## Swarm Stack Deploy

**When to use**: Small production clusters without Kubernetes.

**How**:
1. `docker swarm init` on manager; `docker swarm join` on workers
2. Create overlay network in compose file
3. `docker stack deploy -c compose.yaml <stack>`

**Trade-offs**: Built-in but less ecosystem than Kubernetes.

## Docker Model Runner for Local LLMs

**When to use**: Running LLMs with direct GPU access (containers struggle with driver passthrough).

**How**:
1. Enable DMR in Docker Desktop settings
2. `docker model pull <model>` / `docker model run`
3. Integrate via Compose `provider: type: model` or Open WebUI sidecar

**Trade-offs**: Models run outside container sandbox; simpler hardware access.

## Slim Image Debugging

**When to use**: Production distroless/Alpine images with no shell.

**How**: `docker debug <container>` — ephemeral debug sidecar without modifying image.

**Trade-offs**: Requires Docker Desktop or compatible environment.

## Volume for Persistent State

**When to use**: Databases, uploads, or any data that must survive container replacement.

**How**:
1. `docker volume create <vol>`
2. `docker run -v <vol>:/path/in/container ...`
3. Never rely on container writable layer for important data

**Trade-offs**: Backup/restore planning required; bind mounts trade portability for host path control.
