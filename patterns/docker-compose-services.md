# Docker Compose Services

**Impact:** HIGH  
**Tags:** compose, one-click, supabase, magic-env

## When to use Compose build pack

- One-click services (Supabase, Plausible, etc.)
- Multi-container apps (API + worker + redis)
- Custom `docker-compose.yml` with Coolify "magic" variables

## Magic environment variables

Coolify injects connection strings and hostnames — see docs for current list (`SERVICE_FQDN_*`, `COOLIFY_*`, etc.). Use these instead of hardcoding internal Docker DNS names when possible.

## Self-hosted Supabase on Coolify

Hetzner community tutorial pattern:

1. Deploy Supabase one-click / compose stack
2. Edit compose **before first boot** for pooler tenant ID (`POOLER_TENANT_ID`) — changing later needs migrations
3. Traefik routes for Studio/API — pair with Access at edge
4. Backups via Coolify scheduled DB backup → S3/R2

Hand off RLS/Postgres patterns to `supabase-specialist`.

## Persistent storage

Map volumes in Coolify UI — survives redeploys. Bind mounts vs named volumes: prefer named volumes for databases.

## Pre/post deploy commands

- **Pre**: rarely needed
- **Post**: `prisma migrate`, `alembic upgrade` — idempotent scripts only

## Rolling updates

Enable health checks + rolling update strategy for zero-downtime on stateless web tiers — not for single-replica stateful DBs without failover.

## References

- [Docker Compose (Coolify)](https://coolify.io/docs/knowledge-base/docker/compose)
- [Hetzner Supabase tutorial](https://community.hetzner.com/tutorials/coolify-supabase-deploy/)
