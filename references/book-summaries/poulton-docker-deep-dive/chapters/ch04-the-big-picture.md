# Chapter 4: The Big Picture

## Core Idea

Chapter 4 splits into **The Ops perspective** (pull image → run → exec → stop → rm) and **The Dev perspective** (clone repo → read Dockerfile → build → run custom app). Together they establish the image/container lifecycle and the "containerizing an app" workflow before engine deep dives.

## Frameworks Introduced

- **The Ops perspective / The Dev perspective**: Dual lens—ops cares about runtime objects; dev cares about Dockerfile → image → running app. Read both for full DevOps fluency.
- **Pulling**: Copying an image onto the Docker host (`docker pull`).
- **Containerizing (an app)**: Building application + dependencies into an image via Dockerfile instructions, then running it as a container.
- **Image analogies**: VM templates (ops) or classes (dev)—immutable artifact that instances (containers) launch from.

## Key Concepts

- **Client + engine co-located** on typical installs; `docker version` must show both Client and Server blocks.
- **Images** bundle OS filesystem slice, app, and dependencies—`nginx:latest` includes stripped-down Linux + NGINX.
- **`docker run` flags**:
  - `--name`: container name
  - `-d`: detached background
  - `-p host:container`: port publish (e.g., `8080:80`)
- **`docker exec -it`**: Interactive process inside running container (e.g., `bash`).
- **Minimal images**: Tools like `ps` often absent—by design for size and attack surface; use Docker Debug later for richer inspection.
- **Dockerfile instructions** (psweb example): `FROM`, `LABEL`, `RUN`, `COPY`, `WORKDIR`, `EXPOSE`, `ENTRYPOINT`—each line is a build step.
- **Build context**: `docker build -t name:tag .` requires trailing `.` from directory containing Dockerfile.
- **Multipass access**: Use VM `192.168.x.x:8080` (from `multipass ls` or `ip a | grep 192`) instead of `localhost`.

## Mental Models

- **Image = recipe artifact; container = running instance**—one image, many containers possible.
- **Ops path uses public images**; **Dev path manufactures images** from source—production teams do both daily.
- **Port mapping**: Host `8080` → container `80` lets you run NGINX without binding privileged port 80 on the host.

## Anti-patterns

- **Skipping `docker version` before labs**: Permission denied on Linux means missing `sudo` or group setup—fix before pull/run loops fail mysteriously.
- **Assuming full Linux toolkit in containers**: `command not found` for `ps` is normal—don't rebuild images just to debug with bloated toolchains (use debug tooling when needed).
- **Wrong build directory**: `docker build` without `.` in `psweb/` produces wrong context or failures.
- **localhost on Multipass**: Browser to `localhost:8080` won't hit the VM—use the Multipass IP.

## Code Examples

Ops smoke test:

```bash
docker version
docker images
docker pull nginx:latest
docker images
```

Run, inspect, exec, lifecycle:

```bash
docker run --name test -d -p 8080:80 nginx:latest
docker ps
docker exec -it test bash
ls -l
ps -elf    # often: bash: ps: command not found
exit
docker ps
docker stop test
docker rm test
docker ps -a
```

Dev workflow (psweb):

```bash
git clone https://github.com/nigelpoulton/psweb.git
cd psweb
ls -l
cat Dockerfile
docker build -t test:latest .
docker images
docker run -d \
  --name web1 \
  --publish 8080:8080 \
  test:latest
# Browser: localhost:8080 (Desktop) or 192.168.x.x:8080 (Multipass)
```

Cleanup:

```bash
docker rm web1 -f
docker rmi test:latest
```

Reference Dockerfile structure:

```dockerfile
FROM alpine
LABEL maintainer="nigelpoulton@hotmail.com"
RUN apk add --update nodejs npm curl
COPY . /src
WORKDIR /src
RUN npm install
EXPOSE 8080
ENTRYPOINT ["node", "./app.js"]
```

## Reference Tables

| Command | Ops/Dev | Purpose |
|---------|---------|---------|
| `docker pull` | Ops | Download image to host |
| `docker run` | Both | Create + start container from image |
| `docker ps` / `docker ps -a` | Ops | Running / all containers |
| `docker exec -it` | Ops | Shell into running container |
| `docker stop` / `docker rm` | Ops | Stop and delete container |
| `docker build -t` | Dev | Build image from Dockerfile |
| `docker rmi` | Dev | Remove image |

| Flag | Meaning |
|------|---------|
| `--name` | Container name |
| `-d` | Detached |
| `-p H:C` | Publish host port H to container port C |
| `-it` | Interactive TTY (exec) |
| `-f` | Force (rm running container) |

## Worked Example

Ops: pull `nginx:latest`, run as `test` mapping host 8080→container 80, exec bash and list `/` root filesystem, exit without stopping container, then `stop` + `rm`. Dev: clone `psweb`, build `test:latest` from Dockerfile, run `web1` on 8080, verify Node page in browser, `rm -f` + `rmi` cleanup.

## Key Takeaways

1. Images are pullable artifacts; containers are runnable instances created with `docker run`.
2. Ops fluency: pull → run → ps → exec → stop → rm is the minimum runtime loop.
3. Dev fluency: Dockerfile defines build steps; `docker build -t tag .` then `docker run` containerizes apps.
4. Platform details matter: sudo on Linux, Multipass IP for browser access, minimal tooling inside images by design.

## Connects To

- **Chapter 3**: Working install prerequisite (`docker version` success).
- **Chapter 5**: What the engine does when you run/build.
- **Chapter 6**: Image layers, storage, and pull mechanics beyond `docker images`.
- **Chapter 7**: Container lifecycle management depth.
- **Chapter 8**: Dockerfile instruction reference expands the psweb `FROM`/`RUN`/`ENTRYPOINT` pattern.
- **Chapter 9**: Multi-container apps via Compose—extends the single-container `web1` pattern.
