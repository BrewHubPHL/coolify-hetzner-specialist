# Chapter 7: Working with Containers

## Core Idea

Containers are run-time instances of images—stateless, ephemeral, immutable, and designed to run a **single process** (PID 1). They virtualize the OS (not hardware like VMs), sharing the host kernel for size, speed, and density advantages. Each container gets a thin read-write layer atop the read-only image; lifecycle operations (run, exec, stop, restart, delete) are managed via the Docker CLI against the OCI-compliant Engine stack.

## Frameworks Introduced

- **OS virtualization vs hardware virtualization**: Hypervisor carves CPU/RAM into VMs each with full OS; container runtime carves process trees/filesystems into containers sharing one kernel.
- **The VM tax**: Every VM needs its own OS (patch, boot, resource overhead); containers avoid duplicate OSes.
- **Image R/W layer model**: Shared read-only image + per-container thin read-write layer; stop preserves R/W; delete destroys it.
- **Three ways to start apps**: `ENTRYPOINT` (CLI args append, not override), `CMD` (overridden by CLI args), or explicit `docker run ... <command>`.
- **Restart policy matrix**: `no`, `on-failure`, `always`, `unless-stopped`—controls auto-restart on exit, failure, manual stop, and daemon restart.
- **Docker Debug**: Injects Nix-based toolbox (`/nix`) into slim containers/images for troubleshooting without baking tools into the image.

## Key Concepts

- Containers: smaller, faster, more portable than VMs; microservices = one container per service.
- **Immutable deployment**: Replace failed containers; don't patch live containers in production.
- **PID 1 rule**: Main app process is PID 1; killing it kills the container; container runs only while PID 1 runs.
- **docker version**: Validates Client + Server (Engine) connectivity; Linux needs `docker` group or `sudo` for `/var/run/docker.sock`.
- **docker run flags**: `-d` detached, `--name`, `-p host:container`, `--rm` auto-remove on exit, `--restart`.
- **docker exec**: `-it` interactive shell session vs remote one-shot command execution.
- **docker stop**: SIGTERM to PID 1, 10s grace, then SIGKILL; exit code 137.
- **docker attach** vs **exec**: attach connects to main process; exec starts new process.
- **Ctrl-PQ**: Detach without killing attached process.
- **Docker Debug**: Pro/Team/Business subscription; changes to **running** containers persist; debugging **images/stopped** containers uses sandbox—changes lost on exit; tools never become part of image.
- Compose restart: `restart_policy.condition: always | unless-stopped | on-failure`.

## Mental Models

- **Stopped container vs image**: Image = class; container = instance with optional ephemeral state in R/W layer.
- **Single-process container**: If PID 1 exits, the "room is empty"—container stops.
- **SSH-like exec**: `docker exec -it` ≈ remote shell into running process namespace.
- **Restart policies as self-healing knobs**: `always` even resurrects after `docker stop` when daemon restarts; `unless-stopped` respects manual stop.

## Anti-patterns

- **Editing live containers in production** (vi inside running webserver): Works for demos but violates immutability—build new image, replace container instead.
- **Multi-process containers**: Violates single-service design; obscures health and signal handling.
- **Assuming slim images include shell/vim/ping**: `docker exec` fails if binary missing—use Docker Debug or rebuild with tools only in debug stages.
- **Force-deleting images in use** (`docker rmi -f`): Leaves dangling untagged images.
- **Deleting containers before images**: Delete containers first, then images.

## Code Examples

Check Engine:

```bash
$ docker version
```

Start container with port map:

```bash
$ docker run -d --name webserver -p 5005:8080 nigelpoulton/ddd-book:web0.1
```

Verify:

```bash
$ docker images
$ docker ps
```

Inspect Entrypoint:

```bash
$ docker inspect nigelpoulton/ddd-book:web0.1 | grep Entrypoint -A 3
"Entrypoint": [
"node",
"./app.js"
],
```

Override with CLI (Cmd-style):

```bash
$ docker run --rm -d alpine sleep 60
```

Interactive exec:

```bash
$ docker exec -it webserver sh
/src # ls -l
/src # exit
```

Remote exec:

```bash
$ docker exec webserver ps
```

Inspect container:

```bash
$ docker inspect webserver
```

Lifecycle:

```bash
$ docker stop webserver
$ docker ps -a
$ docker restart webserver
$ docker exec webserver cat views/home.pug
$ docker rm webserver -f
```

PID 1 demo:

```bash
$ docker run --name ddd-ctr -it ubuntu:24.04 bash
root@d3c892ad0eb3:/# ps
root@d3c892ad0eb3:/# exit
$ docker ps -a
$ docker restart ddd-ctr
$ docker attach ddd-ctr
root@d3c892ad0eb3:/# <Ctrl-PQ>
$ docker ps
```

Docker Debug:

```bash
$ docker login
$ docker info
$ docker debug ddd-ctr
docker > ping nigelpoulton.com
docker > install bind
docker > nslookup nigelpoulton.com
docker > exit
$ docker debug nigelpoulton/ddd-book:web0.1
docker > entrypoint --print
node ./app.js
docker > exit
```

Restart policy demo:

```bash
$ docker run --name neversaydie -it --restart always alpine sh
/ # exit
$ docker ps
$ docker inspect neversaydie | grep RestartCount
"RestartCount": 1,
```

Cleanup:

```bash
$ docker rm $(docker ps -aq) -f
$ docker rmi $(docker images -q)
```

## Reference Tables

| VMs | Containers |
|-----|------------|
| Hardware virtualization | OS virtualization |
| Full OS per VM | Shared host kernel |
| Larger, slower boot | Smaller, faster start |
| Dedicated kernel per VM | Shared kernel (harden with seccomp, AppArmor, SELinux) |
| Stateful, migratable | Stateless, ephemeral |

| Restart policy | Non-zero exit | Zero exit | docker stop | Daemon restart |
|----------------|---------------|-----------|-------------|----------------|
| no | N | N | N | N |
| on-failure | Y | N | N | Y |
| always | Y | Y | N | Y |
| unless-stopped | Y | Y | N | N |

| Command | Purpose |
|---------|---------|
| `docker run` | Create and start container |
| `docker ps [-a]` | List running (or all) containers |
| `docker exec [-it]` | Run command in running container |
| `docker stop` | Graceful stop (SIGTERM → SIGKILL) |
| `docker restart` | Restart stopped container |
| `docker rm [-f]` | Delete container |
| `docker attach` | Attach to main process |
| `docker inspect` | Container/image metadata |
| `docker debug` | Debug slim images/containers (subscription) |

| Start app method | CLI override behavior |
|------------------|----------------------|
| ENTRYPOINT | CLI args appended |
| CMD | CLI args replace |
| CLI command | Required if no ENTRYPOINT/CMD |

## Worked Example

`docker run -d --name webserver -p 5005:8080 nigelpoulton/ddd-book:web0.1`: daemon pulls if missing → containerd/runc creates container → Entrypoint `node ./app.js` runs as PID 1 → browser hits localhost:5005. `docker exec -it webserver sh` for troubleshooting; edit `views/home.pug` persists in R/W layer across stop/restart but is lost on `docker rm`. `docker run --restart always alpine sh` + `exit` → Docker restarts same container (`RestartCount: 1`). `docker debug` injects ping/vim via `/nix` without modifying image.

## Key Takeaways

1. Containers = image + thin R/W layer; image stays read-only.
2. One main process (PID 1); its death stops the container.
3. ENTRYPOINT/CMD/CLI define startup; ENTRYPOINT cannot be overridden by CLI args alone.
4. Changes to R/W layer survive stop/restart but not container deletion.
5. Don't mutate production containers—replace with new ones.
6. Docker Debug bridges slim-image operational gaps (subscription required).
7. Restart policies provide local self-healing; `always` vs `unless-stopped` differs on daemon restart after manual stop.
8. Containers win on density and speed; mature platforms mitigate shared-kernel security concerns.

## Connects To

- **Chapter 5**: Every `docker run` traverses daemon → containerd → runc → shim.
- **Chapter 6**: Image layers shared; container adds writable layer; can't delete image while containers reference it.
- **Chapter 8**: ENTRYPOINT/CMD defined in Dockerfile; production images specify startup commands.
- **Chapter 9 (book)**: Compose `restart_policy` extends per-container restart concepts.
- **Coolify/Hetzner**: Port mapping, health via PID 1, immutability, and restart policies map directly to production service definitions.
