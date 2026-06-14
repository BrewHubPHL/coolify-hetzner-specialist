# Chapter 9: Multi-container apps with Compose

## Core Idea

Docker Compose lets you define an entire multi-container microservices application in a declarative YAML Compose file and manage its full lifecycle with `docker compose` sub-commands—replacing brittle shell scripts and long `docker run` chains.

## Frameworks Introduced

- **Compose file as application contract**: The Compose file documents every service, image, port, network, and volume; treat it as code in version control to bridge dev and ops.
  - When to use: Any app with two or more containers that must start, network, and persist data together.
  - How: Name the file `compose.yaml` (or `compose.yml`); use top-level `services`, `networks`, and `volumes` keys.

- **Project naming convention**: Compose prefixes all resources with the build-context directory name (project name) plus service name.
  - When to use: Predicting container, network, and volume names when debugging or scripting.
  - How: `{project}-{service}-{replica}` for containers; `{project}_{resource}` for networks and volumes.

- **Desired lifecycle via compose sub-commands**: `up` deploys; `stop`/`restart` preserve containers and data; `down` removes containers and networks but retains volumes and images by default.
  - When to use: Day-to-day dev workflows where you need fast iteration without losing Redis counters or rebuilt images.
  - How: `docker compose up --detach` → work → `docker compose down` (data persists) → `docker compose up --detach` again.

## Key Concepts

- **Compose**: Docker's integrated tool (formerly Fig) for multi-container apps; reference implementation of the open Compose Specification.
- **Compose file**: YAML defining `services` (mandatory), `networks`, `volumes`, and optional secrets/config.
- **Service**: A logical microservice; containers inherit the service name in their container names.
- **deploy.replicas**: Number of identical containers for a service (port bindings limit replicas on Docker Desktop).
- **build vs image**: `build: .` builds from local Dockerfile; `image:` pulls a pre-built image from a registry.
- **Published vs target ports**: `ports` maps `published` (host) to `target` (container) ports.
- **Project name**: Derived from the directory containing the Compose file (e.g. `multi-container`).

## Mental Models

- **Think of Compose as "docker run for the whole app"**: One file replaces dozens of manual commands for networks, volumes, builds, and starts.
- **Use `down` vs `stop` based on intent**: `stop` pauses without deleting; `down` tears down runtime objects but keeps volumes/images unless you pass `--volumes` / `--rmi`.
- **Compose changes require redeploy**: Editing the Compose file while stopped does not apply on `restart`; run `up` again to reconcile.
- **Volumes survive `down` by design**: Persistent data (Redis counts) outlives container and network deletion.

## Anti-patterns

- **Imperative-only management without a Compose file**: Shell scripts drift from reality and cannot serve as documentation.
- **Expecting `restart` to pick up Compose file edits**: Changes only apply after `docker compose up` redeploys the stack.
- **Running multiple replicas with conflicting host port bindings**: Only one container can bind the same published port on the host.
- **Using `docker-compose` (v1 hyphenated CLI)**: Modern Docker uses `docker compose` (space, v2 integrated plugin).

## Code Examples

```yaml
services:
  web-fe:
    deploy:
      replicas: 1
    build: .
    command: python app/app.py
    ports:
      - target: 8080
        published: 5001
    networks:
      - counter-net
  redis:
    image: "redis:alpine"
    deploy:
      replicas: 1
    networks:
      - counter-net
    volumes:
      - type: volume
        source: counter-vol
        target: /data
networks:
  counter-net:
volumes:
  counter-vol:
```

- **What it demonstrates**: Two-service counter app—web front-end built locally, Redis with persistent volume, shared bridge network.

```bash
docker compose version
git clone https://github.com/nigelpoulton/ddd-book.git
cd ddd-book/multi-container/
docker compose up --detach
docker compose ps
docker compose top
docker compose stop
docker compose restart
docker compose ls
docker compose down
docker compose down --volumes --rmi all
docker compose -f apps/ddd-book/sample-app.yml up --detach
```

- **What it demonstrates**: Full deploy, inspect, stop/restart, list projects, teardown, and custom Compose file path.

```python
cache = redis.Redis(host='redis', port=6379)
```

- **What it demonstrates**: Service-name DNS on the Compose network—`redis` resolves to the redis service containers.

## Reference Tables

| Resource type | Logical name | Example Compose-created name |
|---------------|--------------|------------------------------|
| Service (web-fe) | web-fe | multi-container-web-fe-1 |
| Service (redis) | redis | multi-container-redis-1 |
| Network | counter-net | multi-container_counter-net |
| Volume | counter-vol | multi-container_counter-vol |

| Command | Effect |
|---------|--------|
| `docker compose up [--detach]` | Build/pull images, create networks/volumes, start all services |
| `docker compose stop` | Stop containers; retain containers and data |
| `docker compose restart` | Restart stopped containers; does not apply Compose file changes |
| `docker compose ps` | List containers, commands, status, ports |
| `docker compose top` | Processes inside each container (host PIDs) |
| `docker compose down` | Remove containers and networks; keep volumes and images |
| `docker compose down --volumes --rmi all` | Also remove volumes and all images |

## Worked Example

Deploy the book's `multi-container` sample from `ddd-book/multi-container/compose.yaml`:

1. Clone repo and enter build context: `git clone https://github.com/nigelpoulton/ddd-book.git && cd ddd-book/multi-container/`
2. Deploy: `docker compose up --detach` — builds `multi-container-web-fe`, pulls `redis:alpine`, creates `multi-container_counter-net` and `multi-container_counter-vol`.
3. Verify: `docker compose ps` shows both containers Up; browse `localhost:5001` and refresh to increment the counter stored in Redis on the volume.
4. Tear down partially: `docker compose down` — containers and network removed; `docker volume ls` still shows `multi-container_counter-vol`.
5. Redeploy: `docker compose up --detach` — counter continues from previous value because volume persisted.
6. Full cleanup: `docker compose down --volumes --rmi all`.

## Key Takeaways

1. Store Compose files in Git—they are the authoritative app definition for dev and ops.
2. Project name comes from the directory name and prefixes every created resource.
3. `docker compose up` is the primary deploy command; use `-f` for non-default file paths and `--detach` for background mode.
4. `down` preserves volumes and images by default—intentional data safety for stateful services.
5. Service names become DNS names on Compose-defined networks.
6. Compose Specification is community-led; Docker Compose is the reference implementation—read [compose-spec.io](https://www.compose-spec.io/) for details.

## Connects To

- **Ch 8 (networks/volumes)**: Compose files declare the same networks and volumes you learned to create imperatively.
- **Ch 10 (Docker and AI)**: Compose `provider` extension integrates Docker Model Runner as a service.
- **Ch 12 (Docker Swarm)**: Swarm reuses Compose file format via `docker stack deploy`.
- **Compose Specification**: Open standard for multi-container app definitions beyond Docker alone.
