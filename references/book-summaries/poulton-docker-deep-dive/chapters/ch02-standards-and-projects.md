# Chapter 2: Docker and Container-Related Standards and Projects

## Core Idea

"Docker" means both **Docker, Inc.** (the company) and the **Docker platform** (CLI + engine). The ecosystem is stabilized by **OCI** low-level specs, matured by **CNCF** hosted projects, and extended by **Moby** composable building blocks—Docker implements all three OCI specs and embeds CNCF tech like containerd and Notary.

## Frameworks Introduced

- **Docker platform vs Docker, Inc.**: Platform = technology stack; Inc. = company behind it—use precise language in architecture docs and procurement.
- **CLI + Engine (client/server)**: CLI translates friendly commands to API calls; engine runs server-side container lifecycle—can be co-located or remote.
- **Rail tracks analogy (OCI)**: Standardized "track gauge" for images, runtimes, and distribution—enables interchangeable tools without competing low-level formats.
- **CNCF maturity phases**: Sandbox → Incubating → Graduated—use graduation as a production-readiness signal for hosted projects.
- **Moby Project**: Community toolbox for platform builders to mix Moby, in-house, and third-party components.

## Key Concepts

- **dotCloud origin**: PaaS company with in-house "Docker" loader tool; 2013 pivot to Docker, Inc. and container focus.
- **Engine complexity hidden by CLI**: Daemon orchestrates many subsystems (Figure 2.2)—operators issue `docker` commands; internals abstracted.
- **OCI specs** (vendor-neutral, Linux Foundation):
  - **image-spec**: Image format
  - **runtime-spec**: Container runtime behavior
  - **distribution-spec**: Registry/protocol behavior
- **appc/rkt history**: CoreOS appc competed with Docker; OCI formation archived appc and unified standards—**competing standards slow adoption**.
- **CNCF**: Hosts Kubernetes, containerd, Notary, Prometheus, Cilium, etc.—provides governance, docs, community maturation—not spec authorship like OCI.
- **Moby members**: Microsoft, Mirantis, Nvidia, etc.—Docker platform itself is composed from Moby + CNCF + OCI pieces.

## Mental Models

- **OCI = interchange standards** (tracks); **CNCF = project nursery/incubator** (trains, stations, signaling vendors); **Moby = parts catalog for custom platforms**.
- **Docker as integrator**: Modern Docker is OCI-compliant end-to-end (BuildKit images, OCI runtime, Hub distribution).

## Anti-patterns

- **Conflating Docker with the only runtime**: Production K8s paths often use containerd directly—still OCI-compatible, not "non-Docker."
- **Ignoring OCI when building tooling**: Custom builders/registries/runtimes that violate OCI specs fragment the ecosystem.
- **Treating CNCF Sandbox as production-default**: Check maturity phase before betting critical infrastructure on young projects.
- **Saying "Docker invented containers"**: Docker democratized; Linux kernel features and prior art pre-exist—credit OCI/CNCF for ongoing governance.

## Code Examples

No CLI commands—architectural relationships:

```
docker CLI  --API-->  dockerd (engine/daemon)
                         ├── BuildKit → OCI-compliant images
                         ├── OCI-compliant runtime → containers
                         └── (uses) containerd, Notary [CNCF]
Docker Hub → OCI distribution-spec registry
```

## Reference Tables

| Body | Role | Key outputs |
|------|------|-------------|
| **OCI** | Low-level standards governance | image-spec, runtime-spec, distribution-spec |
| **CNCF** | Cloud-native project hosting | Kubernetes, containerd, Notary, Prometheus, Cilium |
| **Moby** | Platform component community | Specialized tools for custom container platforms |
| **Docker, Inc.** | Product company | Docker Desktop, Engine, Hub, commercial offerings |

| Historical fork | Resolution |
|-----------------|------------|
| Docker vs appc/rkt | OCI formed; appc archived |

## Worked Example

You pull `nginx:latest` from Docker Hub (OCI distribution-spec), the engine creates an OCI-compliant container via its runtime, and the same image runs unchanged on a Kubernetes node using containerd—because all players implement the same "rail gauge."

## Key Takeaways

1. Separate Docker the company from Docker the platform in conversations and design docs.
2. OCI defines the three specs that make images, runtimes, and registries interoperable.
3. CNCF matures projects through Sandbox/Incubating/Graduated—not the same job as OCI.
4. Moby supplies composable parts; Docker's platform is a curated assembly of ecosystem pieces.

## Connects To

- **Chapter 4**: `docker pull` hits an OCI-compliant registry; `docker run` uses an OCI runtime.
- **Chapter 5**: Engine architecture deep dive behind the CLI/daemon split introduced here.
- **Chapter 6**: BuildKit and image-spec details.
- **Chapter 1**: containerd named as Kubernetes' stripped-down Docker lineage.
