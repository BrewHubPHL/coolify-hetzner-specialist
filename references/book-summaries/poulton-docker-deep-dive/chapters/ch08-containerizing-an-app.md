# Chapter 8: Containerizing an App

## Core Idea

**Containerization** packages an application as an OCI image and runs it as a container: write app → create **Dockerfile** → **docker build** → (optional) **docker push** → **docker run**. Modern Docker uses **Buildx** (client) and **BuildKit** (server) by default. Production images stay small via **multi-stage builds**, **build cache** optimization, and slim base images—BuildKit/buildx also enable **multi-architecture** builds locally (QEMU) or via **Docker Build Cloud** (native hardware, paid).

## Frameworks Introduced

- **Five-step containerization flow**: App + dependencies → Dockerfile → build → push (optional) → run.
- **Build context**: Directory (trailing `.`) containing source, dependency manifests, and Dockerfile—sent to builder.
- **Layer vs metadata instructions**: `FROM`, `RUN`, `COPY`, `WORKDIR` add content/layers; `EXPOSE`, `ENV`, `CMD`, `ENTRYPOINT` add metadata (WORKDIR may create small layer on current BuildKit).
- **Multi-stage builds**: Multiple `FROM` = build stages; intermediate images discarded; `COPY --from=<stage>` extracts artifacts into final slim stage (often `scratch`).
- **Build targets**: `--target <stage>` builds specific final stage from one Dockerfile.
- **Buildx / BuildKit / builders**: Buildx CLI plugin → builder (BuildKit instance) via drivers (`docker-container` local + QEMU, `cloud` for Build Cloud).
- **Build cache invalidation**: Cache miss on any instruction invalidates cache for all subsequent lines; order Dockerfile from stable → volatile.

## Key Concepts

- **docker init**: Analyzes build context; generates Dockerfile, `.dockerignore`, `compose.yaml` (Docker Desktop; optional elsewhere).
- **docker build -t name:tag .**: BuildKit parses Dockerfile top-to-bottom; comments start with `#`.
- **docker tag**: Additional name for same image ID (required before push to user/repo on Docker Hub).
- **docker push**: Uses tag to determine registry/repository; layers mount from library if shared.
- **Official / Verified Publisher base images**: Preferred for security and maintenance.
- **Multi-stage parallelism**: Stages with no mutual `FROM` dependency run in parallel (e.g. `build-client` and `build-server` after `base`).
- **scratch**: Empty base for minimal production images containing only binaries.
- **--platform=linux/amd64,linux/arm64**: Multi-arch build; `--push` exports to registry; `--load` loads locally (single arch).
- **Cache hits**: Builder reuses layer if same base + same instruction; `COPY`/`ADD` checksum file content—changed files invalidate cache.
- **--no-cache**: Force full rebuild.
- **npm ci --omit=dev**, **RUN --mount=type=cache**: Production dependency install patterns in generated Dockerfiles.
- **Non-root USER**: Run app as `node` user in production Dockerfile pattern.

## Mental Models

- **Dockerfile as recipe, layers as ingredients**: Each content-adding step stacks; metadata labels the dish (ports, startup command).
- **Multi-stage as factory + shipping box**: Factory stage (golang:alpine, 350MB+) compiles; shipping box (scratch, ~27MB) holds only binaries.
- **Build cache as assembly line**: Break one step → rebuild everything downstream; put changing steps (COPY app code) late.
- **Buildx as foreman, BuildKit as factory floor**: Client sends Dockerfile + context; builder exports image or pushes manifest list.

## Anti-patterns

- **Big images in production**: Including compilers, shells, build tools in final stage—increases size, vulnerabilities, attack surface.
- **Wrong tag push to Docker Hub**: `ddd-book:ch8.node` without username fails push—must `docker tag` to `user/ddd-book:tag`.
- **Cache-unfriendly Dockerfile order**: `COPY .` before dependency install → every code change invalidates dependency layers.
- **Installing "the entire internet"**: Accept all recommended packages (`apt` without `--no-install-recommends`) bloats images.
- **Assuming WORKDIR never creates layers**: Current BuildKit may create small WORKDIR layers—behavior may change.

## Code Examples

Clone app:

```bash
$ git clone https://github.com/nigelpoulton/ddd-book.git
$ cd ddd-book/node-app
$ ls -l
```

Generate Dockerfile (Docker Desktop):

```bash
$ docker init
? What application platform does your project use? Node
? What version of Node do you want to use? 23.3.0
? Which package manager do you want to use? npm
? What command do you want to use to start the app? node app.js
? What port does your server listen on? 8080
```

Generated Dockerfile pattern:

```dockerfile
ARG NODE_VERSION=20.8.0
FROM node:${NODE_VERSION}-alpine
ENV NODE_ENV production
WORKDIR /usr/src/app
RUN --mount=type=bind,source=package.json,target=package.json \
--mount=type=bind,source=package-lock.json,target=package-lock.json \
--mount=type=cache,target=/root/.npm \
npm ci --omit=dev
USER node
COPY . .
EXPOSE 8080
CMD node app.js
```

Build and inspect:

```bash
$ docker build -t ddd-book:ch8.node .
$ docker images
$ docker inspect ddd-book:ch8.node
$ docker history ddd-book:ch8.node
```

Push workflow:

```bash
$ docker login
$ docker tag ddd-book:ch8.node nigelpoulton/ddd-book:ch8.node
$ docker images
$ docker push nigelpoulton/ddd-book:ch8.node
```

Run container:

```bash
$ docker run -d --name c1 \
-p 5005:8080 \
nigelpoulton/ddd-book:ch8.node
$ docker ps
```

Multi-stage Dockerfile (Go example):

```dockerfile
FROM golang:1.23.4-alpine AS base
WORKDIR /src
COPY go.mod go.sum .
RUN go mod download
COPY . .
FROM base AS build-client
RUN go build -o /bin/client ./cmd/client
FROM base AS build-server
RUN go build -o /bin/server ./cmd/server
FROM scratch AS prod
COPY --from=build-client /bin/client /bin/
COPY --from=build-server /bin/server /bin/
ENTRYPOINT [ "/bin/server" ]
```

Build multi-stage:

```bash
$ docker build -t multi:full .
$ docker images
$ docker history multi:full
```

Build targets:

```bash
$ docker build -t multi:client --target prod-client -f Dockerfile-final .
$ docker build -t multi:server --target prod-server -f Dockerfile-final .
$ docker images
```

Buildx builders:

```bash
$ docker buildx ls
$ docker buildx inspect cloud-nigelpoulton-ddd
$ docker buildx create --driver=docker-container --name=container
$ docker buildx use container
```

Multi-arch build:

```bash
$ docker buildx build --builder=container \
--platform=linux/amd64,linux/arm64 \
-t nigelpoulton/ddd-book:ch8.1 --push .
```

Build Cloud:

```bash
$ docker buildx create --driver cloud nigelpoulton/ddd
$ docker buildx build \
--builder=cloud-nigelpoulton-ddd \
--platform=linux/amd64,linux/arm64 \
-t nigelpoulton/ddd-book:ch8.1 --push .
```

Cache demo Dockerfile:

```dockerfile
FROM alpine
RUN apk add --update nodejs npm
COPY . /src
WORKDIR /src
RUN npm install
EXPOSE 8080
ENTRYPOINT ["node", "./app.js"]
```

Force no cache:

```bash
$ docker build --no-cache -t myapp .
```

Cleanup:

```bash
$ docker rm c1 -f
$ docker rmi \
multi:full multi:client multi:server ddd-book:ch8.node nigelpoulton/ddd-book:ch8.node
```

## Reference Tables

| Instruction | Creates layer? | Role |
|-------------|----------------|------|
| FROM | Pulls base layers | Base image / new stage |
| RUN | Yes | Execute build commands |
| COPY | Yes | Add files from context |
| WORKDIR | Often small layer | Set working directory |
| EXPOSE | Metadata | Document port |
| ENV | Metadata | Environment variable |
| CMD | Metadata | Default command (overridable) |
| ENTRYPOINT | Metadata | Fixed entry (args append) |
| USER | Metadata | Run-as user |

| Component | Role |
|-----------|------|
| Buildx | CLI build client (default since Docker v23.0) |
| BuildKit | Build engine/server |
| docker-container driver | Local BuildKit in container; QEMU multi-arch |
| cloud driver | Docker Build Cloud; native AMD/ARM; shared cache |

| Multi-stage stage | Purpose |
|-------------------|---------|
| base | Tooling + dependencies |
| build-client / build-server | Compile binaries (parallel) |
| prod / prod-client / prod-server | Minimal runtime image |

| Flag | Purpose |
|------|---------|
| `-t` | Tag image |
| `-f` | Dockerfile path |
| `--target` | Build specific stage |
| `--platform` | Multi-arch platforms |
| `--push` | Push to registry |
| `--load` | Load single arch locally |
| `--no-cache` | Ignore build cache |

## Worked Example

Clone `ddd-book/node-app` → `docker init` generates Alpine-based Node Dockerfile with cache mounts and non-root user → `docker build -t ddd-book:ch8.node .` creates 7 layers (4 from base + 3 new) → `docker tag` + `docker push` to Hub → `docker run -d --name c1 -p 5005:8080 nigelpoulton/ddd-book:ch8.node` → test localhost:5005. For Go: multi-stage build compiles in `golang:1.23.4-alpine`, copies binaries to `scratch` (~26.7MB vs 350MB+ build image). `docker buildx build --platform=linux/amd64,linux/arm64 --push .` publishes manifest list to Hub.

## Key Takeaways

1. Containerization = Dockerfile + build context + build + run (+ optional push).
2. Buildx/BuildKit is default; understand builders, drivers, and Build Cloud tradeoffs.
3. Layer-creating vs metadata-only instructions explain image size and `docker history`.
4. Multi-stage builds discard build tooling; final image contains only runtime artifacts.
5. `--target` splits one Dockerfile into multiple production images.
6. Order Dockerfile for cache hits: stable dependency installs before `COPY` application code.
7. Multi-arch builds need buildx + `--platform`; registry stores manifest list under one tag.
8. Use Official Images, slim runtimes, and minimal packages in production stages.

## Connects To

- **Chapter 5**: BuildKit is an Engine component; builds produce OCI images consumed by containerd.
- **Chapter 6**: Layers, digests, manifest lists, registries, and `docker push` mechanics.
- **Chapter 7**: `CMD`/`ENTRYPOINT` from Dockerfile drive container PID 1; port `-p` maps host to `EXPOSE`d port.
- **Chapter 9 (book)**: Docker Compose multi-container apps extend single-container patterns.
- **Coolify/Hetzner**: Production deploys use built images from registries; multi-arch and cache strategy matter for CI/CD on heterogeneous servers.
