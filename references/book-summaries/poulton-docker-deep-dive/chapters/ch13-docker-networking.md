# Chapter 13: Docker Networking

## Core Idea

Docker networking is built on the Container Network Model (CNM) implemented by libnetwork and extended by pluggable drivers. Single-host bridge networks suit local dev; port mappings are a clunky escape hatch for external access; MACVLAN connects containers to physical VLANs; and libnetwork provides network-scoped DNS service discovery plus Swarm ingress load balancing.

## Frameworks Introduced

- **Container Network Model (CNM)**: Design spec with three building blocks — **sandboxes** (isolated network stacks inside containers), **endpoints** (virtual NICs connecting sandboxes to networks), and **networks** (virtual 802.1d bridges grouping endpoints).
  - When to use: Any time you reason about how containers get IPs, routes, and isolation.
  - How: Each container gets a sandbox with one or more endpoints; each endpoint attaches to exactly one network; containers on the same network can communicate; endpoints on different networks cannot talk to each other even inside the same container.

- **Libnetwork control plane vs driver data plane**: Libnetwork implements CNM, management APIs, service discovery, and ingress load balancing; **drivers** (bridge, overlay, macvlan) implement network creation, isolation, and connectivity.
  - When to use: Diagnosing which layer owns a behavior (DNS vs VXLAN vs Linux bridge).
  - How: `docker network create -d <driver>` invokes the driver; a single host can run heterogeneous networks owned by different drivers.

- **Single-host bridge network**: Local-scope 802.1d bridge; default `bridge` network maps to kernel `docker0`; custom bridges create `br-<network-id-prefix>` in the host kernel.
  - When to use: Local development and very small apps on one host.
  - How: `docker network create -d bridge <name>`; attach with `--network`; inspect with `brctl show` and `ip link show docker0`.

- **Network-scoped service discovery**: Embedded Docker DNS registers names from `--name` / `--net-alias`; resolution only works among containers on the **same** network.
  - When to use: Inter-container communication by hostname on custom bridge networks.
  - How: Containers forward lookups to embedded DNS; **default `bridge` network does NOT support DNS** — use user-defined bridge networks for name resolution.

- **Ingress vs host publish mode** (Swarm only): Ingress (default) routes external traffic to any node via the ingress overlay mesh; host mode only delivers traffic to nodes running replicas.
  - When to use: Publishing Swarm services externally.
  - How: `--publish published=5005,target=80` = ingress; `--publish published=5005,target=80,mode=host` = host mode.

## Key Concepts

- **Sandbox**: Container's isolated network stack (interfaces, ports, routing, DNS).
- **Endpoint**: Virtual Ethernet interface; one network per endpoint.
- **docker0**: Kernel bridge backing the default `bridge` network (typically 172.17.0.0/16).
- **veth pair**: Virtual cable — one end on bridge, one end in container sandbox.
- **Port mapping / publish**: `-p` / `--publish` maps host port to container port for external access.
- **MACVLAN driver**: Gives containers real MAC/IP on external VLAN; requires NIC promiscuous mode (blocked on most public clouds).
- **Ingress network**: Hidden Swarm overlay network; every node attached; enables routing mesh for published services.
- **docker_gwbridge**: Appears on Swarm members; connects to gateway/overlay plumbing.

## Mental Models

- **Think of networks as virtual switches**: Endpoints plug containers into switches; switches isolate traffic groups.
- **Use custom bridge networks for DNS**: Default bridge is legacy — user-defined bridges get automatic name resolution.
- **Port mappings are a dev hack, not a scale pattern**: Each host port is exclusive; prefer overlay + ingress or proper load balancers for production.
- **MACVLAN = container as physical server**: No port mapping or extra bridges, but promiscuous mode is the gatekeeper.

## Anti-patterns

- **Relying on default bridge for multi-container apps**: No DNS on built-in `bridge`; creates brittle IP-based wiring.
- **MACVLAN on public cloud**: Promiscuous mode is normally blocked — design will fail at deploy time.
- **MACVLAN without IP range reservation**: Driver has no management plane to detect IP conflicts on the physical network.
- **Publishing every service via host port mappings at scale**: Port exhaustion and no shared ingress path.

## Code Examples

```bash
# Create user-defined bridge and test DNS
docker network create -d bridge localnet
docker run -d --name c1 --network localnet alpine sleep 1d
docker run -it --name c2 --network localnet alpine sh
# inside c2: ping c1  # works — embedded DNS

# Port mapping for external access
docker run -d --name web --network localnet --publish 5005:80 nginx
docker port web
# 80/tcp -> 0.0.0.0:5005

# MACVLAN to VLAN 100 (datacenter only)
docker network create -d macvlan \
  --subnet=10.0.0.0/24 --ip-range=10.0.0.0/25 --gateway=10.0.0.1 \
  -o parent=eth0.100 macvlan100

# Swarm ingress vs host publish
docker service create --name svc1 \
  --publish published=5005,target=80,mode=host nginx

# Custom DNS for external lookups
docker run -it --name custom-dns \
  --dns=8.8.8.8 --dns-search=nigelpoulton.com alpine sh
```

- **What it demonstrates**: Bridge creation, name-based discovery, port publish syntax, MACVLAN parent interface, Swarm publish modes, and `--dns` overrides.

## Reference Tables

| Command | Purpose |
|---------|---------|
| `docker network ls` | List networks |
| `docker network create -d bridge` | User-defined bridge |
| `docker network inspect` | Subnet, containers, driver options |
| `docker network prune` / `rm` | Remove unused / specific networks |
| `brctl show` | Kernel bridges and connected veths |
| `ip link show docker0` | Bridge interface state |
| `docker port <container>` | Show port mappings |

| Network driver | Scope | Typical use |
|----------------|-------|-------------|
| `bridge` | Single host | Dev, local services |
| `host` | Single host | Container uses host network stack |
| `none` | — | No networking |
| `macvlan` | Single host | Attach to physical VLAN |
| `overlay` | Swarm | Multi-host (Ch 14) |

| Swarm publish mode | External access | Load balancing |
|--------------------|-----------------|----------------|
| Ingress (default) | Any swarm node | Yes — routing mesh |
| Host | Only nodes with replicas | No |

## Worked Example

**Bridge + port mapping lab**: Create `localnet`, attach `c1` and `c2`, confirm `ping c1` from `c2`. Run `web` with `--publish 5005:80` on `localnet`. From a `client` on default `bridge`, `curl <host-ip>:5005` reaches NGINX — proving host port redirect works but requires knowing host IP and dedicates port 5005. Inspect with `brctl show` to see `veth` on `br-f918f1bb0602` after container attach.

**Service discovery flow**: `c1` pings `c2` → local resolver cache miss → recursive query to embedded DNS → DNS returns `c2` IP (only if same network) → ICMP to resolved IP. Cross-network names return no IP.

## Key Takeaways

1. CNM = sandboxes + endpoints + networks; libnetwork implements it; drivers build topologies.
2. User-defined bridge networks get DNS; default `bridge` does not — always prefer custom bridges.
3. Each custom bridge = new kernel `br-*` bridge; containers connect via veth pairs.
4. Port mappings (`-p host:container`) expose container ports on the host but don't scale and bind host ports exclusively.
5. MACVLAN puts containers on real VLANs with real MACs — powerful in DC, blocked on public cloud.
6. Swarm ingress mode publishes services on all nodes via the ingress overlay; host mode restricts to replica nodes.
7. Troubleshoot connectivity via daemon logs (`journalctl -u docker.service`), container logs (`docker logs`), and `debug`/`log-level` in `daemon.json`.

## Connects To

- **Chapter 14**: Overlay networks for multi-host container networking (replaces port-mapping sprawl).
- **Chapter 12**: Swarm services use overlay networks and ingress publishing.
- **traefik-routing.md / Coolify**: Production routing typically uses reverse proxies or platform ingress instead of raw port mappings.
- **Linux bridges**: Underpins all single-host bridge behavior (`docker0`, `br-*`).
