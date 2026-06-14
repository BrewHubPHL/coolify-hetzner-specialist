# Cheatsheet — Docker Deep Dive

## Platform Choice

| Situation | Do | Because |
|-----------|-----|---------|
| Local dev on Mac/Win/Linux laptop | Docker Desktop | Full feature set: Scout, Debug, init, UI, extensions |
| Can't use Desktop | Multipass `docker` image or Linux snap | Missing Scout/Debug/init |
| Production orchestration at scale | Kubernetes (not Swarm alone) | Industry standard; K8s uses containerd |
| Small team, built-in orchestration | Docker Swarm | Lower ops overhead than K8s |

## VM vs Container

| Dimension | VM | Container |
|-----------|-----|-----------|
| OS per instance | Full guest OS | Shares host kernel |
| Boot time | Slow | Fast |
| Density | ~10 per host | ~50 per host (typical ratio in book) |
| Portability | Moderate | High (same kernel family) |

## Engine Stack (request → running container)

```
docker CLI → API → dockerd → containerd → shim → runc → kernel
```

- **runc exits** after start; **shim** becomes parent
- **dockerd restart** does not kill running containers (daemonless design)

## Image vs Container

| | Image | Container |
|---|-------|-----------|
| Mutability | Immutable layers | Writable thin layer on top |
| Lifecycle | Built/pulled | Created/started/stopped/deleted |
| Analogy | Class | Instance |

## Networking Decisions

| Need | Network type | Notes |
|------|--------------|-------|
| Single-host dev, name resolution | User-defined **bridge** | Default `bridge` has **no DNS** |
| External access (dev) | **Port mapping** `-p` | One host port = one service |
| Multi-host Swarm services | **Overlay** | VXLAN; optional encryption |
| Physical VLAN IP/MAC | **MACVLAN** | Needs promiscuous NIC (rare on public cloud) |
| Swarm external traffic | **Ingress** publish mode | Routing mesh to any node |

## Storage Decisions

| Data type | Use | Avoid |
|-----------|-----|-------|
| DB / state | **Named volume** | Container writable layer |
| Config from host | **Bind mount** | Copying secrets into image |
| Ephemeral scratch | Container layer | Volume for anything important |

## Dockerfile Production Rules

1. **Multi-stage** when build tools ≠ runtime needs
2. **Pin base images** by digest in production
3. **Run as non-root** where possible
4. **One process per container** (PID 1 = your app or proper init)
5. **Scan with Scout** before deploy
6. **`.dockerignore`** to shrink context and cache invalidation

## Restart Policies

| Policy | Use when |
|--------|----------|
| `no` | Batch jobs, one-shot |
| `on-failure` | Retry transient errors |
| `always` | Must survive daemon reboot |
| `unless-stopped` | Like always but respects manual stop |

## Security Quick Checks

- Drop **capabilities**; avoid `--privileged`
- Enable **seccomp** profile (default on Docker)
- Use **read-only** rootfs where feasible
- **Secrets** via Swarm secrets or external vault — not env in image
- **DCT** for signed images in regulated environments

## Smells

| If you see… | You're probably… |
|-------------|------------------|
| Hard-coded container IPs | On default bridge or skipping custom network |
| 2GB+ production images | Missing multi-stage or wrong base |
| `latest` in production | Unreproducible deploys |
| Data lost after `docker rm` | No volume for persistent paths |
| Can't `exec` into prod container | Good (slim image) — use `docker debug` |
