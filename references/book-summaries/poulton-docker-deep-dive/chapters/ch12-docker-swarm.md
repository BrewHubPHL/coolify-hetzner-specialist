# Chapter 12: Docker Swarm

## Core Idea

Docker Swarm groups Docker nodes into a secure, highly-available cluster (swarm) and orchestrates multi-service apps declaratively via Compose files—scheduling replicas across nodes, self-healing, rolling updates, and overlay networking—though Kubernetes has largely superseded it for large-scale production.

## Frameworks Introduced

- **Swarm dual nature**: (1) secure cluster of Docker nodes (swarm, lowercase) and (2) intelligent application orchestrator (Swarm, capital S).
  - When to use: Small businesses needing orchestration without Kubernetes complexity; learning orchestration concepts before Kubernetes.
  - How: `docker swarm init` on managers; workers/managers join with tokens; deploy stacks with `docker stack deploy`.

- **Desired state reconciliation**: Compose file defines desired state; Swarm continuously compares observed cluster state and reconciles differences (replicas, images, networks).
  - When to use: Every Swarm deployment and update.
  - How: Edit Compose file → `docker stack deploy -c compose.yaml <stack>`; Swarm schedules, scales, and updates to match.

- **Declarative management over imperative**: All changes via Compose file + `stack deploy`; imperative commands (e.g. `docker service scale`) create drift that a later deploy will undo.
  - When to use: Production Swarm ops; version-control Compose files as source of truth.
  - How: Change `deploy.replicas`, `image`, `update_config` in file; redeploy—never rely on ad-hoc `service scale` alone.

## Key Concepts

- **Manager node**: Runs control plane (Raft consensus); default also runs user apps—dedicate managers in busy production.
- **Worker node**: Runs user application replicas.
- **Leader / Reachable**: Active/passive HA among managers; one Leader, others Reachable backups.
- **Stack**: Swarm application deployed from a Compose file via `docker stack deploy`.
- **Service**: Swarm object managing one or more identical containers (replicas) across the cluster.
- **Overlay network**: Multi-host network; `driver: overlay` with optional `encrypted: 'yes'` (~10% performance penalty).
- **docker stack deploy**: Deploy and update Swarm apps; creates/updates services, networks, volumes.

## Mental Models

- **Think Kubernetes-lite with Compose files**: Same declarative YAML mental model; smaller ecosystem, gentler learning curve.
- **Desired state is the Compose file on the manager**: Observed state must match; redeploy reconciles everything—including undoing imperative scale changes.
- **Three managers across AZs for HA**: One manager failure tolerated; this is cluster HA—not automatic app HA if only one DB replica existed on the failed node.
- **Stack name prefixes resources**: `ddd_web-fe`, `ddd_counter-net`, `ddd_counter-vol` for stack `ddd`.

## Anti-patterns

- **Imperative `docker service scale` without updating Compose**: Next `stack deploy` reverts replica count to file definition.
- **Running production workloads on all managers**: Dedicate managers to control plane under load.
- **Using Docker Desktop for multi-node Swarm labs**: Single node only—use Play with Docker, Multipass, or VMs.
- **Expecting `stack rm` to delete volumes**: Volumes decoupled from service lifecycle; manual `docker volume rm` required.
- **Choosing Swarm for large vibrant ecosystems**: Kubernetes won the orchestration wars for most production scale.

## Code Examples

```bash
docker swarm init
docker swarm join --token SWMTKN-1-... 172.31.40.192:2377
docker swarm join-token manager
docker node ls
```

- **What it demonstrates**: Initialize swarm, add workers and additional managers, list nodes with Leader/Reachable status.

```yaml
networks:
  counter-net:
    driver: overlay
    driver_opts:
      encrypted: 'yes'
volumes:
  counter-vol:
services:
  web-fe:
    image: nigelpoulton/ddd-book:swarm-app
    command: python app.py
    deploy:
      replicas: 4
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
    networks:
      - counter-net
    ports:
      - "5001:8080"
  redis:
    image: "redis:alpine"
    networks:
      - counter-net
    volumes:
      - type: volume
        source: counter-vol
        target: /data
```

- **What it demonstrates**: Swarm stack with encrypted overlay, rolling update policy, restart policy, four web-fe replicas, persistent Redis volume.

```bash
cd ddd-book/swarm-new
docker stack deploy -c compose.yaml ddd
docker stack ps ddd
docker stack services ddd
docker stack ls
docker stack rm ddd
docker volume rm ddd_counter-vol
docker swarm leave
docker swarm leave --force
```

- **What it demonstrates**: Deploy counter app, inspect replicas across nodes, list stacks, remove stack, cleanup volume, leave swarm.

Declarative scale + image update (edit Compose then redeploy):

```yaml
services:
  web-fe:
    image: nigelpoulton/ddd-book:swarm-appv2
    deploy:
      replicas: 10
```

```bash
docker stack deploy -c compose.yaml ddd
docker stack ps ddd
```

- **What it demonstrates**: Scale to 10 replicas and roll to new image; Swarm adds six new replicas immediately, updates original four two-at-a-time per `update_config`.

## Reference Tables

| Node role | MANAGER STATUS column | Typical workload |
|-----------|----------------------|------------------|
| Manager (leader) | Leader | Control plane + optional apps |
| Manager (backup) | Reachable | Control plane + optional apps |
| Worker | (empty) | User application replicas |

| Swarm port | Purpose |
|------------|---------|
| 2377 | Cluster management (must be open between nodes) |

| Command | Purpose |
|---------|---------|
| `docker swarm init` | Create new swarm; node becomes first manager |
| `docker swarm join` | Add worker or manager with token |
| `docker swarm join-token manager` | Print command to add managers |
| `docker node ls` | List nodes, roles, Leader/Reachable |
| `docker stack deploy -c <file> <name>` | Deploy or update stack from Compose file |
| `docker stack ls` | List stacks and service counts |
| `docker stack ps <name>` | Per-replica node, image, desired/current state |
| `docker stack services <name>` | Service mode, replica count, ports |
| `docker stack rm <name>` | Delete stack (no confirmation prompt) |
| `docker swarm leave [--force]` | Remove node from swarm |

| `deploy.update_config` field | Book example value | Effect |
|------------------------------|-------------------|--------|
| `parallelism` | 2 | Update two replicas at a time |
| `delay` | 10s | Wait between update batches |
| `failure_action` | rollback | Revert on failed update |

| `deploy.restart_policy` field | Book example value | Effect |
|-------------------------------|-------------------|--------|
| `condition` | on-failure | Restart only failed replicas |
| `delay` | 5s | Pause between restart attempts |
| `max_attempts` | 3 | Limit restart retries |
| `window` | 120s | Stop retrying after two minutes |

## Worked Example

Five-node swarm (3 managers, 2 workers) deploying `ddd-book/swarm-new`:

1. On first manager: `docker swarm init` — note worker join token.
2. On workers: paste `docker swarm join --token ...` from init output.
3. On manager: `docker swarm join-token manager` — run join command on two more nodes for HA managers.
4. `docker node ls` — verify Leader + 2 Reachable managers + 2 workers.
5. `git clone ... && cd ddd-book/swarm-new`
6. `docker stack deploy -c compose.yaml ddd` — creates overlay network, volume, `ddd_web-fe` (4 replicas), `ddd_redis` (1 replica).
7. `docker stack services ddd` — confirm `4/4` web-fe replicas, `*:5001->8080`.
8. Browse any node IP on port 5001 — counter app works across cluster.
9. Edit Compose: `image: swarm-appv2`, `replicas: 10`; `docker stack deploy -c compose.yaml ddd`.
10. `docker stack ps ddd` — observe old replicas Shutdown, new ones Running with v2 image; rolling update honors parallelism/delay.
11. Cleanup: `docker stack rm ddd`; `docker volume rm ddd_counter-vol` on redis node; `docker swarm leave --force` on all nodes (leader last).

## Key Takeaways

1. Swarm = secure multi-node cluster + Compose-driven orchestrator; good for small scale and Kubernetes learning path.
2. Always manage declaratively—imperative scale changes drift from desired state and get reverted on next deploy.
3. Production: three managers across availability zones; dedicate managers from app workloads under load.
4. `docker stack deploy` is deploy-and-update; Compose `deploy` block controls replicas, updates, and restart behavior.
5. Encrypted overlay networks protect traffic but cost ~10% performance—test your workload.
6. `stack rm` removes services and networks but not volumes; swarm is declining vs Kubernetes for most new projects.

## Connects To

- **Ch 9 (Compose)**: Same YAML format; Swarm adds `deploy` orchestration keys and overlay networks.
- **Ch 13+ (overlay networking)**: Deep dive on overlay networks used by Swarm multi-host apps.
- **Kubernetes**: Poulton positions Swarm as stepping stone to Quick Start Kubernetes / The Kubernetes Book.
- **Coolify / production orchestration**: Modern self-hosted stacks typically use Swarm-compatible Compose or Kubernetes—not bare single-node Compose.
