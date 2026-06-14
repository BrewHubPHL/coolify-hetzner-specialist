---
name: coolify-hetzner-specialist
description: Coolify PaaS on Hetzner VPS — provisioning, Traefik, Docker Compose services, Doppler secrets, backups, tunnels, and fleet onboarding for self-hosted deploys.
version: "1.0.0"
tags:
  - coolify
  - hetzner
  - docker
  - traefik
  - self-hosted
  - paas
when_to_use:
  - Provisioning or hardening a Hetzner box for Coolify
  - Deploying apps, databases, or one-click services via Coolify
  - Configuring Traefik routes, health checks, or rolling updates
  - Wiring Doppler secrets at container boot (DOPPLER_WRAP pattern)
  - Setting up backups (S3/R2), snapshots, or fleet machine onboarding
  - Integrating Coolify with Cloudflare Tunnels (hand off to cloudflare-specialist for Access)
invocation_examples:
  - "Onboard a new fleet server — Coolify + tunnel + healthz checklist"
  - "Fix Traefik 502 on a Coolify-deployed Python app"
  - "Configure scheduled Postgres backups to R2"
  - "Clone a brain app into a queue drainer worker via APP_COMMAND"
disable-model-invocation: true
---

# Coolify + Hetzner Specialist

Self-hosted PaaS patterns for **Coolify** on **Hetzner Cloud** (and similar VPS). Aligns with sovereign infrastructure: no incumbent PaaS lock-in, you own the box, backups, and tunnels.

## Overview

Coolify turns SSH-accessible Linux servers into a deployment plane: Git → build (Nixpacks/Dockerfile/Compose) → Traefik → HTTPS. Hetzner provides cost-effective EU/US VPS with snapshots and API provisioning.

### Progressive disclosure map

| Need | Load |
|------|------|
| Install, servers, resources | `patterns/coolify-fundamentals.md` |
| Hetzner API / sizing / snapshots | `patterns/hetzner-provisioning.md` |
| Traefik, domains, health checks | `patterns/traefik-routing.md` |
| Compose stacks, magic env vars | `patterns/docker-compose-services.md` |
| Doppler at boot | `patterns/doppler-secrets.md` |
| Backups S3/R2, instance backup | `patterns/backups-dr.md` |
| New fleet machine runbook | `patterns/fleet-onboarding.md` |
| Mistakes | [anti-patterns.md](anti-patterns.md) |
| Abstract product wiring | [examples/brew-hub-integration.md](examples/brew-hub-integration.md) |
| Links | [references/official-docs-links.md](references/official-docs-links.md) |

### Retrieval rule

Coolify UI and API evolve quickly. Prefer [Coolify docs](https://coolify.io/docs) and Hetzner docs over memory for install commands and API paths.

---

## Core Principles

### 1. SSH is the only onboarding requirement

Coolify manages remote servers over SSH. Hetzner, bare metal, Pi, ThinkPad — same model.

### 2. Traefik is the default door on the box

Apps get FQDNs in Coolify; Traefik terminates TLS (or pairs with Cloudflare edge TLS). Origin config matters for tunnel routes — see `cloudflare-specialist` `zero-trust-tunnels.md`.

### 3. Secrets never live in git

Use Doppler (`doppler run` at entrypoint), Coolify encrypted env, or CI-injected build vars — not committed `.env` files.

### 4. Docker bypasses host ufw

Published container ports may be reachable on public interfaces. **Real** exposure control: no gateway port-forwards + Cloudflare Tunnel outbound-only pattern.

### 5. Claim Coolify immediately after install

Unclaimed instances get picked up by internet scanners. Set root user on first boot.

---

## BrewHub philosophy

- **Debt-free self-hosting**: Hetzner VPS + Coolify vs per-seat SaaS PaaS
- **Kill switches**: don't expose admin UIs without Access; don't skip snapshots before brain cutovers
- **Thriving wages as infrastructure**: runbooks (`fleet-onboarding.md`) over hero SSH

See [examples/brew-hub-integration.md](examples/brew-hub-integration.md).

---

## Integration with Other Specialists

| Skill | Handoff |
|-------|---------|
| `cloudflare-specialist` | cloudflared systemd, Access apps, tunnel route recipes |
| `supabase-specialist` | Self-hosted Supabase compose on Coolify; pooler config |
| `python-ai-agents-specialist` | `entrypoint.sh`, `APP_COMMAND`, gunicorn workers |
| `nextjs-specialist` | Next.js on Coolify vs OpenNext on Workers — choose per surface |
| `raspberry-pi-specialist` | Smaller fleet nodes; memory limits |

---

## Book-to-skill

`references/book-summaries/` — one chapter per file. Validate with `node scripts/validate-skill.mjs`.
