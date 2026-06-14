# Chapter 14: Docker Overlay Networking

## Core Idea

Overlay networks create flat, secure layer-2 networks spanning multiple Docker hosts via VXLAN tunnels over the underlay (physical/IP) network. Containers on different hosts communicate directly as if on one switch — complexity hidden behind `docker network create -d overlay`. Overlay requires Swarm mode and is the foundation of cloud-native multi-host microservices.

## Frameworks Introduced

- **Overlay network**: Swarm-scoped virtual L2 network built by the native overlay driver on libnetwork; containers on different hosts share one subnet and communicate without host port mappings.
  - When to use: Multi-host microservices that need direct container-to-container connectivity across nodes.
  - How: `docker network create -d overlay <name>` on a manager; attach Swarm services with `--network`; use `--attachable` for standalone containers.

- **Underlay vs overlay**: **Underlay** = existing L3 infrastructure (routers, physical networks); **overlay** = virtual L2 tunneled through underlay via VXLAN; underlay sees ordinary UDP/IP packets.
  - When to use: Explaining overlay to networking teams or debugging cross-host connectivity.
  - How: Overlay abstracts topology — a 10.0.0.0/24 flat network can span hosts on different physical subnets.

- **Lazy overlay extension**: Worker nodes only receive overlay network plumbing when a container/service needs it — reduces gossip and improves scalability.
  - When to use: Understanding why `docker network ls` on a worker may not show an overlay until a replica is scheduled there.
  - How: First service replica on a worker triggers network extension to that node.

- **VXLAN tunnel + VTEP model**: Each host gets a sandbox with virtual switch `Br0`, a VTEP (one end on `Br0`, one end on host stack bound to UDP **4789**), and a VXLAN tunnel between VTEPs.
  - When to use: Deep troubleshooting of cross-host packet flow.
  - How: VTEP encapsulates frames in UDP with VNID header; remote VTEP de-encapsulates and forwards to local `Br0` and container veth.

- **Encrypted overlay**: `-o encrypted` encrypts data plane with TLS (AES-GCM); keys rotate every 12 hours; ~10% performance penalty.
  - When to use: Production overlays where application traffic must be encrypted on the wire.
  - How: Control plane is always encrypted; data plane encryption is optional via `-o encrypted`.

## Key Concepts

- **VXLAN**: Encapsulation technology creating L2 networks atop L3 infrastructure; transparent to routers.
- **VTEP (VXLAN Tunnel Endpoint)**: Encapsulates/de-encapsulates overlay traffic; bound to UDP port 4789.
- **VNID (VXLAN Network ID)**: Maps VLANs to VXLAN segments; maintains isolation on de-encapsulation.
- **Br0**: Virtual switch inside per-host network sandbox for overlay traffic.
- **docker_gwbridge**: Bridge connecting overlay to external/gateway paths on Swarm nodes.
- **Scope `swarm`**: Overlay networks span entire cluster, not single host.
- **Gossip protocol**: Propagates container endpoint info to all swarm nodes for ARP proxy behavior.

## Mental Models

- **Overlay = virtual switch stretched across hosts**: Containers plug into `Br0` locally; VXLAN is the cable between hosts.
- **Underlay complexity is opaque**: Containers see one hop to remote peers — traceroute shows single hop across overlay.
- **Swarm is prerequisite**: Overlay leverages cluster store, security, and key-value state — not available in standalone Docker.
- **Don't boast VXLAN expertise to network teams**: Know enough to collaborate; the tunnel still traverses real infrastructure.

## Anti-patterns

- **Overlay without Swarm**: Overlay driver requires swarm mode — won't work on standalone nodes.
- **Standalone containers on overlay without `--attachable`**: Default overlay only accepts Swarm services.
- **Forcing encrypted overlay without measuring impact**: ~10% throughput hit — test before mandating in production.
- **Inspecting overlay on workers before scheduling**: Network may not exist until a replica lands on the node.

## Code Examples

```bash
# Swarm prerequisites — open between nodes:
# 2377/tcp (management), 7946/tcp+udp (gossip), 4789/udp (VXLAN data plane)

docker swarm init                    # on node1
docker swarm join --token <token> <manager-ip>:2377  # on node2

# Encrypted overlay
docker network create -d overlay -o encrypted uber-net

# Service spanning two nodes
docker service create --name test \
  --network uber-net --replicas 2 ubuntu sleep infinity

# Attachable overlay for standalone containers
docker network create -d overlay --attachable my-overlay

# Multi-subnet overlay (L3 routing within overlay)
docker network create \
  --subnet=10.1.1.0/24 --subnet=11.1.1.0/24 \
  -d overlay prod-net
```

- **What it demonstrates**: Swarm init, encrypted overlay creation, multi-replica cross-node service, attachable flag, and dual-subnet overlay.

## Reference Tables

| Port | Protocol | Purpose |
|------|----------|---------|
| 2377 | TCP | Swarm management plane |
| 7946 | TCP + UDP | Control plane gossip (SWIM) |
| 4789 | UDP | VXLAN data plane |

| Overlay option | Effect |
|----------------|--------|
| `-d overlay` | Native overlay driver |
| `-o encrypted` | Encrypt data plane (AES-GCM, 12h key rotation) |
| `--attachable` | Allow standalone containers, not just services |
| `--subnet` (multiple) | Multiple L2 segments with internal routing |

| Encryption scope | Default |
|--------------------|---------|
| Control plane | Always encrypted |
| Data plane | Optional (`-o encrypted`) |

## Worked Example

**Two-node overlay ping test**: Manager creates `uber-net` overlay; `test` service with 2 replicas schedules one replica per node — overlay extends to worker on first replica. `docker network inspect uber-net` on node1 shows local replica IP (e.g. 10.0.0.3); same on node2 for remote replica (10.0.0.4). Exec into replica on node1, `ping 10.0.0.4` and `ping test.2.<task-id>` both succeed. `traceroute 10.0.0.4` shows **one hop** — direct overlay path, underlay invisible.

**VXLAN packet walk** (C1 on node1 → C2 on node2 at 10.0.0.4): C1 floods ARP on veth → `Br0` proxy ARP (gossip knew C2's location) → veth forwards to local VTEP → VTEP wraps frame in UDP/4789 with VNID + remote VTEP IP → underlay routes UDP packet → node2 kernel delivers to VTEP on 4789 → de-encapsulate → `Br0` → C2 veth.

## Key Takeaways

1. Overlay networks = flat L2 across hosts; core of cloud-native multi-host Docker apps.
2. Built on VXLAN: VTEPs on each host, UDP 4789, virtual `Br0` switch per host sandbox.
3. Requires Swarm — uses cluster store and security features.
4. Workers lazily extend overlay networks when replicas need them.
5. `-o encrypted` adds data-plane encryption (~10% perf cost); control plane always encrypted.
6. `--attachable` enables standalone containers on overlay; default is services-only.
7. Open 2377, 7946, and 4789 between all swarm nodes before debugging connectivity.

## Connects To

- **Chapter 13**: Bridge networks are single-host; overlay replaces multi-host bridge hacks.
- **Chapter 16**: Overlay networks require Swarm; cluster store encrypted; overlay encryption option.
- **Chapter 12**: Swarm services default to overlay for multi-replica cross-node apps.
- **Kubernetes CNI**: Similar overlay concepts (Flannel, Calico VXLAN) — Docker overlay is Swarm-native equivalent.
