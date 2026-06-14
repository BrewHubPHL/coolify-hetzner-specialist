# Chapter 0: About the Book

## Core Idea

The May 2025 edition is a fast on-ramp to Docker and containers with no prior experience required. It is organized as two sections—**The big picture stuff** then **The technical stuff**—and is updated at least annually because Docker, AI, and the cloud-native ecosystem move too fast for multi-year-old material to stay safe.

## Frameworks Introduced

- **The big picture stuff / The technical stuff**: Two-part book architecture. Use Part 1 when you need vocabulary, motivation, and first hands-on wins; use Part 2 when you need engine internals, production patterns, and advanced topics.
- **Annual edition cadence**: Treat Docker books older than ~2–3 years as dangerously outdated; prefer the latest Nigel Poulton edition when learning or recommending.

## Key Concepts

- Docker changed how we build, share, and run applications and now intersects **Wasm** and **AI**.
- May 2025 adds a full chapter on **Docker Model Runner** for running local LLMs through Docker.
- Kindle readers cannot receive automatic updates; contact Kindle Support or the author (`ddd@nigelpoulton.com`) for edition issues.
- Formats: hardback, paperback, and e-book.

## Mental Models

- **Career lens**: Strong Docker skills map to the best jobs working with the best technologies—containers are baseline infrastructure literacy, not a niche.
- **Book as map**: Chapters 1–4 = orientation; 5–8 = deep technical foundations; 9+ = multi-container apps, AI, Wasm, Swarm, networking, storage, security.

## Anti-patterns

- **Learning from stale Docker books**: Old editions miss Model Runner, current engine behavior, and ecosystem shifts—wastes time and teaches wrong defaults.
- **Assuming Kindle auto-updates**: Do not expect the latest edition on Kindle without manual intervention.
- **Skipping Part 1**: Jumping straight to engine internals without the 30,000-foot view loses context for why containers, standards, and Kubernetes relationships matter.

## Code Examples

No commands in this chapter—it's meta-orientation. First commands appear in Chapter 3 (`docker version`) and Chapter 4 (pull/run/build workflow).

## Reference Tables

| Chapter | Topic |
|---------|-------|
| 1 | History and future of Docker and containers |
| 2 | Container-related standards and projects |
| 3 | Ways to get Docker |
| 4 | Deploy your first container |
| 5 | Docker Engine architecture deep dive |
| 6 | Images and image management |
| 7 | Containers and container management |
| 8 | Containerizing applications |
| 9 | Multi-container AI chatbot with Docker Compose |
| 10 | Local AI models with Docker Model Runner |
| 11 | Build, containerize, and run a Wasm app |
| 12 | Swarm cluster and app deployment |
| 13 | Docker networking deep dive |
| 14 | Overlay networks |
| 15 | Persistent and non-persistent data |
| 16 | Linux and Docker security technologies |

## Worked Example

A developer targeting Coolify/Hetzner work reads Chapter 0, notes Part 1 covers first deploy (Ch 4) before Part 2's engine and networking depth (Ch 5–16), and picks Docker Desktop for local parity with book examples before touching server installs.

## Key Takeaways

1. This edition is current as of May 2025 and includes Docker Model Runner for local LLMs.
2. Structure is deliberately two-phase: big picture first, technical depth second.
3. Annual updates are a feature, not marketing—treat freshness as a learning requirement.
4. Full roadmap spans containers, Compose, AI, Wasm, Swarm, networking, storage, and security.

## Connects To

- **Chapter 1**: Why containers exist and how Docker fits the evolution from bare metal → VMs → containers.
- **Chapter 3**: Docker Desktop as the recommended local install path.
- **Chapter 10**: Docker Model Runner (flagged here, taught in depth later).
- **Coolify/Hetzner context**: Part 2 chapters on networking, data, and security align with production self-hosting patterns.
