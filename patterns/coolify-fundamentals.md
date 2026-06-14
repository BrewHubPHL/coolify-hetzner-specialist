# Coolify Fundamentals

**Impact:** CRITICAL  
**Tags:** coolify, servers, resources, install

## What Coolify manages

| Concept | Meaning |
|---------|---------|
| **Server** | SSH-target Linux host (localhost or remote) |
| **Project / Environment** | Grouping (prod, staging) |
| **Resource** | App, database, or one-click service |
| **Destination** | Docker network target (standalone vs Swarm) |

## Install (self-hosted)

Official path — verify current script at [coolify.io/docs](https://coolify.io/docs/get-started/installation):

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

Post-install:

1. Open dashboard (default `:8000` — tunnel or SSH port-forward only in production)
2. **Create root user immediately** — unclaimed instances are scanned
3. Add SSH private key for remote servers
4. Validate server: Docker running, reachable via SSH

## Build packs

| Pack | Use |
|------|-----|
| **Nixpacks** | Auto-detect Node/Python/etc. |
| **Dockerfile** | Full control (Python ADK, custom entrypoints) |
| **Docker Compose** | Multi-container (Supabase stack, etc.) |
| **Static** | Pre-built assets + Nginx |

## Git deploy flow

1. Connect GitHub/GitLab/Gitea app or deploy key
2. Push to branch → webhook → build → deploy
3. PR preview deployments optional per app

## Scheduled tasks (cron)

Coolify native cron for marketing jobs, reconcilers, queue drainers — prefer over OS cron for visibility. Use [supported cron syntax](https://coolify.io/docs/knowledge-base/supported-cron-syntax) from docs.

## API + MCP

Coolify exposes REST API (Bearer token) and an MCP server for read-only infra visibility — useful for agent debugging.

## Health checks

Configure in UI or Dockerfile `HEALTHCHECK` — required for Traefik routing and rolling updates. 502/503 often = wrong port or app listening on `127.0.0.1` only.

## References

- [Coolify introduction](https://coolify.io/docs/get-started/introduction)
- [Servers](https://coolify.io/docs/knowledge-base/server/introduction)
- [Health checks](https://coolify.io/docs/knowledge-base/health-checks)
