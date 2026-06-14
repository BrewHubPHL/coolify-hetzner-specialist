# Chapter 15: Volumes and Persistent Data

## Core Idea

Containers get ephemeral writable storage tied to their lifecycle; stateful apps need **volumes** — first-class objects decoupled from containers that persist data across container deletion and can be shared via external storage drivers. Treat containers as immutable; never modify live container config — replace with new containers instead.

## Frameworks Introduced

- **Persistent vs non-persistent data**: Persistent = must survive (customer records, DBs, audit logs); non-persistent = scratch/temp. Stateful apps create persistent data; stateless apps don't.
  - When to use: Deciding storage strategy per workload.
  - How: Non-persistent → container thin writable layer; persistent → Docker volumes.

- **Volume lifecycle decoupling**: Volumes are independent API objects managed via `docker volume`; deleting a container does **not** delete its volumes.
  - When to use: Any database, file store, or stateful service.
  - How: Create volume → mount into container at path → data written to mount path lives on volume → delete container → volume and data survive → remount into new container.

- **Volumes vs bind mounts vs tmpfs**: **Volumes** (recommended) — Docker-managed, driver-backed, decoupled lifecycle. **Bind mounts** — map a host path directly into container (`--mount type=bind,source=/host/path,target=/container/path`); no separate volume object; host path must exist; tied to host filesystem layout. **Tmpfs** — in-memory mount for sensitive ephemeral data.
  - When to use: Volumes for production persistent data; bind mounts for dev config injection or when you need a specific host path; avoid bind mounts in portable deploys.
  - How: `--mount source=<vol>,target=/path` for volumes; `--mount type=bind,source=/host,target=/path` for bind mounts. Dockerfile `VOLUME` instruction declares mount points but cannot specify host paths (OS portability).

- **Local driver vs third-party plugins**: Default `local` driver stores under `/var/lib/docker/volumes/<name>/_data` on the host; plugins connect to cloud SAN/NAS/NFS for shared multi-node access.
  - When to use: Single-node = local driver; multi-node shared storage = plugin + external system.
  - How: `docker volume create -d <plugin> <name>`; plugin must be installed first.

- **Shared volume corruption risk**: Multiple containers writing same shared volume without coordination can cause cache/commit races — last writer wins silently.
  - When to use: NFS/LUN presented to multiple nodes.
  - How: Design apps to coordinate writes or use DB-level locking; don't assume shared filesystem = safe concurrent writes.

## Key Concepts

- **Thin writable layer / graphdriver storage**: Ephemeral container layer atop image layers; deleted with container.
- **Mountpoint**: Host path for local volumes (`/var/lib/docker/volumes/<name>/_data`).
- **Scope `local`**: Volume only available to containers on same node as volume.
- **Stateful vs stateless**: Determines whether volumes are mandatory.
- **Container immutability**: Never change live container config; replace container with new image/config.
- **VOLUME Dockerfile instruction**: `VOLUME <container-path>` — declares mount point; host path supplied at deploy time.

## Mental Models

- **Volume = USB drive; container = laptop**: Unplug drive (delete container), data stays on drive (volume).
- **Bind mount = symlink to host folder**: Fast for dev, brittle across hosts/OS — host path must exist and match.
- **Image layers are read-only stack**: Only thin top layer is writable; volumes bypass this for persistent paths.
- **Shared storage needs app-level coordination**: Filesystem sharing ≠ transactional safety.

## Anti-patterns

- **Storing persistent data in container writable layer**: Data lost on `docker rm`.
- **Modifying running container configuration**: Network, app config, packages — always rebuild and replace.
- **Direct host filesystem access to volume data in production**: `/var/lib/docker/volumes/*/_data` bypasses container isolation — demo/education only.
- **`docker volume prune --all` without review**: Deletes all unused volumes — irreversible data loss.
- **Concurrent uncoordinated writes to shared volumes**: Silent data corruption from cached writes.

## Code Examples

```bash
# Create and inspect volume
docker volume create myvol
docker volume inspect myvol
# Mountpoint: /var/lib/docker/volumes/myvol/_data

# Mount volume into container (--mount preferred over legacy -v)
docker run -it --name voltainer \
  --mount source=bizvol,target=/vol \
  alpine
# bizvol auto-created if missing

# Write data, delete container, data persists
docker exec -it voltainer sh -c 'echo "data" > /vol/file1'
docker rm voltainer -f
docker volume ls          # bizvol still exists
cat /var/lib/docker/volumes/bizvol/_data/file1  # host peek (demo only)

# Remount surviving volume in new container
docker run -it --name newctr --mount source=bizvol,target=/vol alpine sh
cat /vol/file1            # data survived

# Bind mount (contrast — host path required)
docker run -it --mount type=bind,source=/host/config,target=/etc/app/config alpine sh

# Dockerfile declares volume point (no host path)
# VOLUME /data

# Cleanup
docker volume rm bizvol   # fails if in use
docker volume prune       # deletes ALL unused — caution
```

- **What it demonstrates**: Volume lifecycle independence, `--mount` syntax, auto-create on run, persistence across container deletion, bind mount alternative, and safe deletion rules.

## Reference Tables

| Storage type | Lifecycle | Host path | Multi-node | Production use |
|--------------|-----------|-----------|------------|----------------|
| Container writable layer | With container | graphdriver | No | Scratch/temp only |
| Volume (`local`) | Independent | `/var/lib/docker/volumes/` | No (single node) | Persistent data |
| Volume (plugin) | Independent | External system | Yes (shared) | Cluster stateful apps |
| Bind mount | With container | User-specified | No | Dev config, host tools |

| Command | Purpose |
|---------|---------|
| `docker volume create` | New volume (`-d` for driver) |
| `docker volume ls` | List volumes |
| `docker volume inspect` | Mountpoint, driver, scope |
| `docker volume rm` | Delete specific unused volume |
| `docker volume prune` | Delete all unused volumes |

| `--mount` fields | Meaning |
|------------------|---------|
| `source` | Volume name or host path (bind) |
| `target` | Path inside container |
| `type` | `volume` (default), `bind`, `tmpfs` |

## Worked Example

**bizvol persistence walkthrough**: `docker run -it --name voltainer --mount source=bizvol,target=/vol alpine` — Docker auto-creates `bizvol`. Write `file1` inside `/vol`. `docker rm voltainer -f` — container gone, `docker volume ls` still shows `bizvol`. Host `ls /var/lib/docker/volumes/bizvol/_data/` confirms file. New container `newctr` with same `--mount source=bizvol,target=/vol` — `cat /vol/file1` returns original content. Proves volume decoupling. Attempt `docker volume rm bizvol` while mounted → error "volume is in use".

## Key Takeaways

1. Ephemeral writable layer dies with container — never store important data there.
2. Volumes are first-class objects: create, inspect, delete independently of containers.
3. `--mount source=vol,target=/path` — missing volume auto-created on run.
4. Local driver: `/var/lib/docker/volumes/<name>/_data`; scope `local` = single node only.
5. Bind mounts map host paths directly — useful for dev, risky for portable production deploys.
6. Third-party volume plugins enable shared storage across cluster nodes — requires coordination to avoid corruption.
7. Treat containers as immutable — replace, don't modify live containers.
8. `docker volume prune` is destructive — prefer `docker volume rm` for precision.

## Connects To

- **Chapter 9**: Compose `volumes:` blocks declare and mount volumes in multi-container apps.
- **Chapter 12**: Swarm services can mount volumes per replica — shared volumes need external drivers.
- **backups-dr.md**: Volume backup strategies depend on driver and mount paths.
- **Coolify**: Platform manages volume persistence for deployed services on Hetzner hosts.
