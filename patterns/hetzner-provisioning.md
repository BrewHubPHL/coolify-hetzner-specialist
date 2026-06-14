# Hetzner Provisioning

**Impact:** HIGH  
**Tags:** hetzner, vps, snapshots, api

## Why Hetzner

Cost-effective VPS with EU/US locations, snapshot API, and a Coolify marketplace image option.

## Create server (console or API)

Hetzner provides a **Coolify image** with Docker preinstalled — see [Hetzner Coolify docs](https://docs.hetzner.com/cloud/apps/list/coolify/).

API example (from Hetzner docs):

```bash
curl -X POST \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-coolify-server","server_type":"cpx22","image":"coolify"}' \
  'https://api.hetzner.cloud/v1/servers'
```

Coolify also has **Create Hetzner Server** in its API — token validated against Hetzner before store.

## Sizing guidance

| Workload | Starting point |
|----------|----------------|
| Coolify control plane + 2–3 small apps | CPX22 (2 vCPU, 4GB) |
| Python brain + MCP + workers | CPX32+ or dedicated box |
| Shop box + self-hosted Supabase | Memory-heavy — 8GB+ |
| Pi / edge | See `raspberry-pi-specialist` |

Profile with Sentinel metrics in Coolify before downsizing.

## Snapshots (mandatory before cutovers)

Take manual snapshot **immediately before**:

- Python brain / MCP cutover
- Postgres major upgrade
- Traefik/proxy migration

Restore = Hetzner snapshot rollback + verify Coolify `APP_KEY` / DB consistency per [backup/restore Coolify instance](https://coolify.io/docs/knowledge-base/how-tos/backup-and-restore-coolify) docs.

## Firewall reality

Hetzner cloud firewall + ufw on host — but **Docker published ports bypass ufw**. Do not rely on ufw alone; use tunnel-only exposure for admin surfaces.

## SSH access

- Key-based auth only
- Prefer Tailscale SSH for admin + tunnel for public hostnames
- Coolify needs SSH from control plane to managed servers

## References

- [Hetzner Coolify app](https://docs.hetzner.com/cloud/apps/list/coolify/)
- [Load balancing on Hetzner (Coolify)](https://coolify.io/docs/knowledge-base/how-tos/load-balancing-on-hetzner)
