# Chapter 16: Docker Security

## Core Idea

Docker security is defense in depth: Linux kernel technologies (namespaces, cgroups, capabilities, MAC, seccomp) with sensible defaults, plus Docker-native tools (Swarm TLS, Docker Scout, Docker Content Trust, secrets). Defaults are moderately secure with zero effort — customize for production but expect significant testing effort.

## Frameworks Introduced

- **Defense in depth layers**: Namespaces (isolation) → cgroups (resource limits) → capabilities (least privilege) → MAC/AppArmor/SELinux (mandatory access) → seccomp (syscall filtering) → Docker Scout (image vulns) → DCT (image signing) → Swarm TLS/secrets (runtime trust).
  - When to use: Security architecture reviews and hardening checklists.
  - How: Each layer adds constraints; defaults cover most apps; customize capabilities/seccomp/MAC for high-security workloads.

- **Namespaces as isolation (not strong boundary)**: Six namespaces per container — pid, net, mnt, ipc, user, uts. Containers look like isolated OSes but are lighter than VMs and **not** as secure as VMs.
  - When to use: Understanding what containers can and cannot see.
  - How: Each namespace virtualizes one OS construct; root namespaces on host remain separate; cross-container access requires network or explicit sharing.

- **Cgroups as resource limits**: Prevent any container from monopolizing shared host CPU, RAM, disk I/O, network I/O — DoS protection.
  - When to use: Multi-tenant hosts, noisy-neighbor prevention.
  - How: Docker applies cgroup limits per container; configure via `--cpus`, `--memory`, etc.

- **Capabilities as least privilege**: Linux root = bundle of ~40+ capabilities; Docker drops most by default and can add back only what's needed (e.g. `CAP_NET_BIND_SERVICE`).
  - When to use: Containers that need some root powers but not full root.
  - How: Start as root, drop all caps, add specific caps; containers cannot re-add dropped caps.

- **seccomp syscall filtering**: Default profile disables ~40–50 of 300+ syscalls; moderate security with wide app compatibility.
  - When to use: Reducing kernel attack surface.
  - How: Default profile applied automatically; custom profiles possible but complex; `--security-opt seccomp=unconfined` disables (avoid in production).

- **Swarm secure-by-default**: Single `docker swarm init` configures cryptographic node IDs, mutual TLS, CA with 90-day cert rotation, encrypted cluster store (etcd), secure join tokens, encrypted overlay networks.
  - When to use: Any Swarm cluster — security is automatic.
  - How: Join tokens are the sole admission gate; rotate with `docker swarm join-token --rotate`; CA expiry via `docker swarm update --cert-expiry`.

- **Docker Scout vulnerability scanning**: Subscription service; builds SBOM from images, compares against CVE databases, ranks severity (C/H/M/L), recommends fixes including updated base images.
  - When to use: Supply chain security in CI and before deploy.
  - How: `docker scout quickview <image>` for summary; `docker scout cves <image>` for detail and remediation links.

- **Docker Content Trust (DCT)**: Cryptographic image signing and verification via Notary; `DOCKER_CONTENT_TRUST=1` enforces signed-only pulls.
  - When to use: Pulling images over untrusted networks; verifying publisher integrity.
  - How: `docker trust key generate` → `docker trust signer add` → `docker trust sign` on push; inspect with `docker trust inspect`.

- **Docker secrets** (Swarm only): Sensitive data encrypted at rest in cluster store, encrypted in flight, mounted as in-memory files to authorized service replicas only; destroyed when replica terminates.
  - When to use: Passwords, TLS certs, API keys in Swarm services.
  - How: `docker secret create` → `docker service create --secret <name>`; least-privilege — only granted services access secrets.

## Key Concepts

- **Root namespaces**: Host's own namespace collection; containers cannot see host processes/files by default.
- **pid namespace**: Isolated process tree; container PID 1; no visibility of host/other container processes.
- **net namespace**: Isolated interfaces, IPs, ports, routing per container.
- **mnt namespace**: Isolated root filesystem; container `/` ≠ host `/`.
- **user namespace**: Map container users to different host users (e.g. container root → unprivileged host user).
- **AppArmor / SELinux**: MAC profiles applied by default per distro; moderately protective, wide compatibility.
- **Cluster store**: Encrypted etcd on managers; holds swarm state, overlay config, secrets.
- **Join tokens**: `SWMTKN-1-<version>-<swarm-id-hash>-<token>` — manager and worker tokens differ only in final field.
- **SBOM**: Software bill of materials from image scan — list of packages for CVE matching.

## Mental Models

- **Containers = efficient but less secure than VMs**: Namespaces isolate but don't harden — layer cgroups, caps, seccomp on top.
- **Capabilities beat naive non-root**: Strip root down to exact permissions needed, not powerless unprivileged users.
- **Join tokens are cluster passwords**: Pattern-match `SWMTKN` in repos; rotate on compromise.
- **Scout awareness = responsibility**: Knowing about CVEs obligates mitigation — scan in CI, fix or accept risk.
- **Secrets in memory, not env vars**: Swarm secrets mount as tmpfs files — better than baking into images or plain env.

## Anti-patterns

- **Treating namespaces as VM-equivalent security**: Containers share kernel — kernel exploit = host risk.
- **Running containers as full root without cap dropping review**: Default is better but custom images may need tuning.
- **Posting Swarm join tokens in repos**: `SWMTKN` prefix is designed for leak detection — still catastrophic if exposed.
- **DCT enabled without signing pipeline**: `DOCKER_CONTENT_TRUST=1` blocks all unsigned pulls — breaks deploys.
- **Secrets outside Swarm mode**: `docker secret` requires swarm — won't work on standalone Docker.
- **Disabling seccomp/AppArmor for convenience**: `--security-opt seccomp=unconfined` removes syscall filtering.
- **Scout as complete security**: Scans images only — not networks, nodes, or orchestrator config.

## Code Examples

```bash
# Swarm secure init (one command = TLS, CA, encrypted store, join tokens)
docker swarm init
docker swarm join-token manager
docker swarm join-token worker
docker swarm join-token --rotate manager   # revoke compromised token

# CA rotation period
docker swarm update --cert-expiry 720h     # 30 days

# Docker Scout scanning
docker scout quickview nigelpoulton/tu-demo:latest
docker scout cves nigelpoulton/tu-demo:latest

# Docker Content Trust workflow
docker trust key generate nigel
docker trust signer add --key nigel.pub nigel myorg/myrepo
docker trust sign myorg/myrepo:v1
export DOCKER_CONTENT_TRUST=1
docker pull myorg/myrepo:v1   # signed — works
docker pull unsigned/image    # fails without trust data

# Docker secrets (Swarm required)
docker secret create db_password ./password.txt
docker service create --name db --secret db_password mysql

# Inspect node TLS cert
sudo openssl x509 -in /var/lib/docker/swarm/certificates/swarm-node.crt -text
```

- **What it demonstrates**: Swarm security bootstrap, token rotation, Scout CVE reporting, DCT sign/verify flow, and secret injection pattern.

## Reference Tables

| Linux technology | Role | Docker default |
|------------------|------|----------------|
| Namespaces | Isolation (pid, net, mnt, ipc, user, uts) | All six per container |
| cgroups | Resource limits (CPU, RAM, I/O) | Applied per container |
| Capabilities | Least-privilege permissions | Sensible subset of root caps |
| AppArmor/SELinux | Mandatory access control | Distro-specific moderate profile |
| seccomp | Syscall allowlist | ~40–50 syscalls blocked |

| Swarm security feature | Auto-configured |
|------------------------|-----------------|
| Cryptographic node IDs | Yes |
| Mutual TLS (mTLS) | Yes |
| CA + cert rotation (default 90d) | Yes |
| Encrypted cluster store | Yes |
| Secure join tokens | Yes |
| Encrypted overlay networks | Yes (control plane always; data plane optional) |

| Docker-native security | Scope | Requires |
|------------------------|-------|----------|
| Docker Scout | Image CVE scanning | Subscription |
| Docker Content Trust | Image signing/verification | Notary, key management |
| Docker secrets | Runtime secret injection | Swarm mode |

| Join token format | Field |
|-------------------|-------|
| `SWMTKN-1-<swarm-id-hash>-<role-token>` | PREFIX-VERSION-SWARM_ID-TOKEN |

## Worked Example

**DCT end-to-end**: Generate key pair (`docker trust key generate nigel`). Associate signer with repo (`docker trust signer add --key nigel.pub nigel myorg/dct`). Sign and push (`docker trust sign myorg/dct:signed`). Inspect signatures (`docker trust inspect myorg/dct:signed --pretty` — shows signer, digest, keys). Enable `export DOCKER_CONTENT_TRUST=1` — unsigned pull fails with "remote trust data does not exist". Delete local image, pull signed image — succeeds because Notary trust data validates publisher and integrity.

**Secret workflow**: Create secret from file → stored encrypted in cluster store → deploy service with `--secret` → Docker encrypts over network to replica → mounts unencrypted file on in-memory filesystem for app use → replica stops → memory filesystem destroyed, secret flushed from node. Other services without grant cannot access.

## Key Takeaways

1. Defense in depth: namespaces + cgroups + capabilities + MAC + seccomp — all with sensible defaults.
2. Namespaces provide isolation, not VM-grade security — augment with additional layers.
3. Capabilities implement least privilege — drop all root caps, add back only what's needed.
4. seccomp default blocks ~40–50 syscalls; customizing is powerful but complex.
5. `docker swarm init` alone delivers TLS, CA rotation, encrypted etcd store, and join tokens.
6. Protect join tokens (`SWMTKN`); rotate on compromise; never commit to repos.
7. Docker Scout scans images for CVEs via SBOM — subscription service; scan ≠ fix obligation.
8. DCT signs/verifies images; `DOCKER_CONTENT_TRUST=1` enforces signed-only pulls.
9. Secrets require Swarm; encrypted at rest and in flight; mounted in-memory to authorized replicas only.

## Connects To

- **Chapter 12**: Swarm mode unlocks secrets, overlay encryption, and secure cluster store.
- **Chapter 14**: Overlay networks depend on Swarm security (encrypted control plane).
- **Chapter 13**: Network namespace isolation per container; ingress TLS on Swarm.
- **fleet-onboarding.md / Coolify**: Production hardening combines Scout scanning, least-privilege caps, and platform-level access control beyond Docker defaults.
