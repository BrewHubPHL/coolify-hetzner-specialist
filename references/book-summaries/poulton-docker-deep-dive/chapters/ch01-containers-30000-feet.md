# Chapter 1: Containers from 30,000 Feet

## Core Idea

Containers are a newer virtualization layer that shares the host OS kernel, packing more workloads per machine than VMs while booting faster and traveling more portably. Docker made Linux containers accessible; today containers dominate cloud-native apps while **Wasm** and **AI** push adjacent innovation.

## Frameworks Introduced

- **The bad old days → Hello VMware! → VMwarts → Hello Containers!**: Evolution narrative—use it to explain why containers won on density and ops overhead, not just hype.
- **Kernel-sharing rule**: Containers share the host kernel—use this to decide compatible host OS (Linux app → Linux kernel; Windows app → Windows kernel).
- **Containerized app**: Terminology for an application running as a container (used throughout the book).

## Key Concepts

- **Pre-VM era**: One app per server, over-provisioned hardware, 5–10% utilization.
- **VM benefits**: Multiple isolated apps per server; **VMwarts**: per-VM OS overhead, patching, monitoring, slow boot, limited portability.
- **Container density**: A host running ~10 VMs might run ~50 containers.
- **Linux foundations**: namespaces, cgroups, capabilities—kernel work Google and others contributed; Docker simplified consumption.
- **Windows**: Supports **Windows containers** (Windows kernel) and **Linux containers** (via **WSL 2** on Win10/11). Almost all real-world and book examples are **Linux containers**.
- **Mac**: No Mac containers; **Docker Desktop**, Podman, or Rancher Desktop run Linux workloads (Docker Desktop uses a lightweight Linux VM).
- **Wasm**: Smaller, faster, more portable binaries via Wasm runtime—but immature standards/ecosystem vs containers; future is VMs + containers + Wasm side-by-side.
- **Docker and AI**: GPU/accelerator passthrough into containers is hard; **Docker Model Runner** runs local LLMs outside containers for direct hardware access (Chapter 10).
- **Kubernetes**: Industry-standard orchestrator; older K8s used Docker, modern K8s uses **containerd**—all Docker containers still work on Kubernetes.

## Mental Models

- **VM = house with its own plumbing (full OS)**; **container = apartment sharing building utilities (shared kernel)**—more units, less duplicate infrastructure.
- **Docker as democratizer**: Pre-Docker containers existed but were too complex for mainstream teams.
- **Ecosystem maturity ladder**: Containers (dominant, rich tooling) → Wasm (emerging) → AI (special-case hardware via Model Runner).

## Anti-patterns

- **Treating Windows and Linux containers as interchangeable**: Kernel mismatch breaks portability—match container type to host kernel (or use WSL 2 for Linux on Windows).
- **Expecting Mac-native containers**: Plan for Linux-in-VM workflows on macOS.
- **Assuming containers solve GPU AI locally**: Use Model Runner when you need direct accelerator access, not default `docker run` assumptions.
- **Ignoring Kubernetes runtime shift**: Know that clusters use containerd under the hood—debug orchestration with that mental model.

## Code Examples

No Docker CLI in this chapter—conceptual only. Runtime relationship:

```
Kubernetes (modern) → containerd → OCI runtime → container
Kubernetes (legacy) → Docker Engine → container
```

## Reference Tables

| Technology | Kernel shared | Typical use in book |
|------------|---------------|---------------------|
| Linux container | Linux | All examples |
| Windows container | Windows | Rare in practice |
| Linux on Windows | Linux (WSL 2) | Dev/test on Windows |
| Mac workloads | Linux (hidden VM) | Docker Desktop daily driver |

| Competitor/alternative | Role vs containers |
|------------------------|-------------------|
| VMware VMs | Heavier isolation, full OS per workload |
| Wasm | Smaller/faster apps, limited ecosystem |
| Kubernetes | Orchestration layer above containers |

## Worked Example

A team on Windows 11 Pro enables WSL 2, installs Docker Desktop in **Linux containers mode**, and develops the same Linux images they'll deploy to Hetzner—avoiding Windows-container-only paths that won't match production.

## Key Takeaways

1. Containers beat VMs on efficiency and portability by sharing the host kernel.
2. Docker made containers mainstream; Linux containers are the default everywhere that matters.
3. Wasm and AI are adjacent forces—containers remain the cloud-native workhorse.
4. Kubernetes orchestrates containers via containerd today; Docker-format images remain portable.

## Connects To

- **Chapter 2**: OCI standards and CNCF projects (containerd, etc.) named here in K8s context.
- **Chapter 3**: Docker Desktop/WSL 2 install paths for Linux containers on dev machines.
- **Chapter 10**: Docker Model Runner for local LLM hardware access.
- **Chapter 11**: Wasm chapter foreshadowed as coexisting with containers.
- **Chapter 12**: Kubernetes/Swarm orchestration threads.
