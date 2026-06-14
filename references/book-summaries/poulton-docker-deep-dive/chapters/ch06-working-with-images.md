# Chapter 6: Working with Images

## Core Idea

An image (Docker image, container image, OCI image) is a read-only package containing everything needed to run an application—code, dependencies, minimal OS filesystem objects, and metadata. Images are build-time constructs; containers are run-time constructs. Docker stacks independent read-only **layers** and presents them as a unified object, enabling sharing, efficient pulls, and multi-architecture delivery behind a single tag.

## Frameworks Introduced

- **Build > share > run pipeline**: Registries sit centrally between building images and running containers.
- **Content-addressable storage**: Every image and layer gets an immutable **digest** (cryptographic hash); tags are mutable pointers.
- **Manifest list + manifest model**: Multi-architecture images hide platform-specific manifests behind one tag; registry API selects the correct manifest for the host platform.
- **Content hash vs distribution hash**: Uncompressed content hash for layer identity; compressed distribution hash for push/pull integrity verification.
- **Local repository / image cache**: On Linux typically `/var/lib/docker/<storage-driver>`; Docker Desktop stores inside the VM.

## Key Concepts

- One image → many containers; image is read-only, bound to running containers until all are deleted.
- **Slim images**: No shell, no package manager if not needed at runtime; Alpine ~3MB; "just enough OS" (no kernel—containers use host kernel).
- **Docker Hub defaults**: Assumes `latest` tag and `docker.io` registry if unspecified.
- **Registry → repository → image/tag** hierarchy; OCI registries implement distribution-spec + Docker Registry v2 API.
- **Official repositories**: Top-level namespace, green Docker Official Image badge, vetted by Docker and vendor.
- **Unofficial repositories**: User/org namespace (e.g. `nigelpoulton/tu-demo`); assume unsafe until verified.
- **Fully qualified name**: `[registry/][namespace/]repository:tag` — Docker fills registry (`docker.io`) and tag (`latest`) defaults.
- **Layers**: Each layer = one or more files; image = manifest listing layers + stack order; **base layer** is first; higher layers obscure files below (copy-on-write semantics at image level).
- **Storage drivers**: `overlay2` default; alternatives `zfs`, `btrfs`, `vfs`—same UX regardless.
- **Layer sharing**: Local and registry deduplication—"Already exists" on pull when layer already present.
- **docker history** vs **docker inspect RootFS**: history shows build steps (some instructions metadata-only); RootFS lists actual layer digests.
- Instructions **ENV, EXPOSE, CMD, ENTRYPOINT** add metadata, not layers (in final image sense per history).
- **Multi-architecture**: `docker buildx imagetools inspect` / `docker manifest inspect` reveal manifest lists; Build Cloud (native, paid) vs emulation via QEMU (local, slow).
- **Docker Scout**: SBOM-based vulnerability scanning (CLI, Desktop, Hub, scout.docker.com); paid subscription.

## Mental Models

- **VM template vs stopped container**: Image ≈ stopped container template; class vs object for developers.
- **Layer stack as transparency sheets**: Upper layer files hide same-path files below; changes = new layers, never in-place edits.
- **Tag as nickname, digest as fingerprint**: `alpine:latest` today ≠ `alpine:latest` last year; `@sha256:...` guarantees identity.
- **Manifest list as menu**: One tag → pick linux/amd64 or linux/arm64 manifest → pull those layers.

## Anti-patterns

- **Trusting `latest` as newest**: `latest` is arbitrary; can point to older image than `v2` tag.
- **Reusing tags for security fixes without versioning**: Pushing fixed `golftrack:1.5` over vulnerable `golftrack:1.5` loses traceability—use new tags or digests.
- **Using unofficial images without verification**: Default-deny for `user/repo` images from the internet.
- **Assuming `docker history` = layer list**: Metadata-only instructions appear; not strict final layer enumeration.
- **Ignoring digest pulls in production**: Mutable tags break reproducible deploys.

## Code Examples

Pull with defaults (latest, Docker Hub):

```bash
$ docker pull redis
Using default tag: latest
latest: Pulling from library/redis
08df40659127: Download complete
4f4fb700ef54: Already exists
<Snip>
Digest: sha256:76d5908f5e19fcdd73daf956a38826f790336ee4707d9028f32b24ad9ac72c08
Status: Downloaded newer image for redis:latest
docker.io/library/redis:latest
```

List local images:

```bash
$ docker images
```

Pull by tag variants:

```bash
$ docker pull redis:8.0-M02
$ docker pull busybox:glibc
$ docker pull alpine
```

Pull from unofficial repo and alternate registry:

```bash
$ docker pull nigelpoulton/tu-demo:v2
$ docker pull ghcr.io/regclient/regsync:latest
```

Inspect layers:

```bash
$ docker pull node:latest
$ docker inspect node:latest
$ docker history node:latest
```

Digests:

```bash
$ docker images --digests alpine
$ docker buildx imagetools inspect nigelpoulton/k8sbook:latest
$ docker pull nigelpoulton/k8sbook@sha256:13dd59a0c74e9a147800039b1ff4d61201375c008b96a29c5bd17244bce2e14b
$ curl "https://hub.docker.com/v2/repositories/nigelpoulton/k8sbook/tags/?name=latest" \
| jq '.results[].digest'
```

Multi-arch inspection:

```bash
$ docker buildx imagetools inspect alpine
$ docker manifest inspect golang | grep 'architecture\|os'
```

Multi-arch build (Build Cloud):

```bash
$ docker buildx build \
--builder=cloud-nigelpoulton-ddd-cloud \
--platform=linux/amd64,linux/arm64 \
-t nigelpoulton/tu-demo:latest --push .
```

Docker Scout:

```bash
$ docker scout quickview nigelpoulton/tu-demo:latest
$ docker scout cves nigelpoulton/tu-demo:latest
```

Delete images:

```bash
$ docker rmi redis:latest af111729d35a sha256:c5b1261d...f8e1ad6b
$ docker rmi $(docker images -q) -f
```

## Reference Tables

| Term | Meaning |
|------|---------|
| Image / OCI image | Read-only app package + metadata |
| Tag | Mutable image name (`repo:tag`) |
| Digest | Immutable content hash (`repo@sha256:...`) |
| Layer | Read-only filesystem diff |
| Manifest | Lists layers for one architecture |
| Manifest list | Index of per-arch manifests |
| Local repository | Cached images on host |
| overlay2 | Default storage driver |

| Command | Purpose |
|---------|---------|
| `docker pull` | Download from registry |
| `docker images [--digests]` | List local images |
| `docker inspect` | Metadata + RootFS layers |
| `docker history` | Build instruction history |
| `docker manifest inspect` | Registry manifest list |
| `docker buildx imagetools inspect` | Manifest/digest queries |
| `docker scout quickview/cves` | Vulnerability scan |
| `docker rmi` | Remove local image |

| Hash type | When used |
|-----------|-----------|
| Content hash | Uncompressed layer identity |
| Distribution hash | Compressed push/pull verification |
| Image digest | Hash of image manifest |

## Worked Example

Pull `redis:latest`: Docker assumes Docker Hub and `latest`; seven layers download but one shows `Already exists` (shared with Portainer extension image). Run `docker inspect node:latest` → `RootFS.Layers` lists five SHA256 layer IDs. For production pinning, `docker buildx imagetools inspect` gets digest, then `docker pull repo@sha256:...` guarantees exact image. On M4 Mac, `docker pull alpine` resolves manifest list → linux/arm64 manifest → pulls arm layers only.

## Key Takeaways

1. Images stack read-only layers; manifests define order and composition.
2. Tags are mutable; digests are immutable—use digests for reproducibility.
3. Layers deduplicate locally and on registries ("Already exists").
4. Multi-arch images use manifest lists; BuildKit/buildx builds and inspects them.
5. Official images are curated but still require caution; unofficial images default to distrust.
6. Docker Scout integrates vulnerability scanning across CLI, Desktop, and Hub.
7. `latest` ≠ newest; never rely on it for production semantics.

## Connects To

- **Chapter 5**: containerd converts Docker images to OCI bundles for runc.
- **Chapter 7**: Containers add thin R/W layer on top of image layers.
- **Chapter 8**: Dockerfiles create layers; BuildKit builds OCI images.
- **Chapter 16 (book)**: Image scanning and slim images reduce attack surface.
- **Coolify/Hetzner**: Registry pulls, digest pinning, and multi-arch images matter for reproducible deploys on ARM vs x64 servers.
