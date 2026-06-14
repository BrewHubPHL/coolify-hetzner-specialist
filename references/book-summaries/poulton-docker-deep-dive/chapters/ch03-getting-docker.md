# Chapter 3: Getting Docker

## Core Idea

Install **Docker Desktop** unless policy blocks it—full engine, UI, Compose, optional Kubernetes, and premium DX plugins (`docker scout`, `docker debug`, `docker init`). Fall back to **Multipass** (VM-based Docker image) or **Linux server install** when Desktop isn't an option, accepting reduced features.

## Frameworks Introduced

- **Docker Desktop-first**: Default learning and daily-dev path for the book and real work.
- **Linux containers mode (Windows)**: Default and required for book examples—switch via whale tray only when you explicitly need Windows containers.
- **Hidden Linux VM (Mac)**: Client native on Darwin; daemon in lightweight Linux VM—explains `darwin/*` client vs `linux/*` server in `docker version`.
- **Multipass docker image**: Pre-baked Ubuntu VM with Docker—use for multi-node lab clusters without Desktop.
- **sudo vs docker group (Linux)**: Server installs often need `sudo` until you add your user to the `docker` group.

## Key Concepts

- **Docker Desktop includes**: Engine, UI, plugins, extension marketplace, Docker Compose, optional local Kubernetes.
- **Licensing**: Free for personal/education; paid if company >250 employees or >$10M revenue.
- **Windows prereqs**: 64-bit Win10/11, BIOS virtualization enabled, **WSL 2** backend.
- **Platform container support**:
  - Win Pro/Enterprise: Windows + Linux containers
  - Mac, Linux, Win Home: Linux containers only
- **Multipass gaps**: No out-of-box `docker scout`, `docker debug`, `docker init`.
- **Linux snap example**: Ubuntu 24.04 `sudo snap install docker`—one of many install methods; verify current docs for production servers.

## Mental Models

- **Desktop = integrated dev appliance**; **Multipass/Linux = engine-only or lab VM**—choose based on feature needs, not familiarity alone.
- **Verify with `docker version`**: Client + Server blocks both present = working install; `OS/Arch` tells you Linux-vs-Windows container mode.

## Anti-patterns

- **Defaulting to Multipass/Linux when Desktop is allowed**: You lose scout/debug/init and the book's intended workflow.
- **Running Windows containers mode for book labs**: Hides Linux containers until you switch back—stay in Linux mode.
- **BIOS changes without care**: Virtualization enablement is required on Windows—treat firmware edits cautiously.
- **Forgetting sudo on fresh Linux installs**: Permission denied on daemon socket → use `sudo` or fix group membership before scripting automation.

## Code Examples

Verify install (Windows—note `linux/amd64` server):

```bash
docker version
```

Mac client vs Linux server architectures:

```bash
docker version
# Client OS/Arch: darwin/arm64 (or darwin/amd64)
# Server OS/Arch: linux/arm64 (or linux/amd64)
```

Multipass workflow:

```bash
multipass launch docker --name node1
multipass ls
multipass shell node1
docker --version
docker info
exit
multipass delete node1
multipass purge
```

Linux snap install and test:

```bash
sudo snap install docker
sudo docker --version
sudo docker info
```

Avoid sudo permanently (Linux):

```bash
sudo groupadd docker
sudo usermod -aG docker $(whoami)
sudo service docker start
```

## Reference Tables

| Install path | Best for | Missing vs Desktop |
|--------------|----------|-------------------|
| **Docker Desktop** | Daily dev, book examples | — (licensing on large orgs) |
| **Multipass + docker image** | No Desktop, multi-node labs | scout, debug, init |
| **Linux server (snap/apt)** | Headless servers | scout, debug, init, UI |

| Platform | Daemon location | Container types |
|----------|-----------------|-----------------|
| Windows + WSL2 | Linux VM/backend | Linux (default) |
| macOS | Hidden Linux VM | Linux only |
| Linux native | Host | Linux |

## Worked Example

On Mac, install Docker Desktop from the official download, start from Launchpad, run `docker version`, confirm Server `OS/Arch` is `linux/arm64`, then proceed to Chapter 4 ops exercises on `localhost`.

## Key Takeaways

1. Docker Desktop is the recommended install—full feature set and book alignment.
2. `docker version` is the smoke test; read Client vs Server `OS/Arch` for platform quirks.
3. Multipass delivers Docker in a VM quickly when Desktop isn't viable—note the 192.168.x.x IP for remote access.
4. Linux server installs work but lack Desktop-only tooling; handle `sudo`/group setup early.

## Connects To

- **Chapter 1**: WSL 2 for Linux containers on Windows; Mac has no native Mac containers.
- **Chapter 4**: First `docker pull`/`docker run` assumes a working Desktop or Multipass install.
- **Chapter 5**: Engine architecture behind the daemon you just installed.
- **Hetzner/Coolify**: Linux server install patterns mirror headless production hosts (without Desktop UI).
