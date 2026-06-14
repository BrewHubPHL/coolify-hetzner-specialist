# Coolify + Hetzner — Navigation

## Pattern priority

| Priority | File |
|----------|------|
| CRITICAL | `fleet-onboarding.md`, `coolify-fundamentals.md` |
| HIGH | `traefik-routing.md`, `doppler-secrets.md`, `backups-dr.md` |
| MEDIUM | `hetzner-provisioning.md`, `docker-compose-services.md` |

## Hetzner + Coolify quick path

```
Hetzner server (Ubuntu LTS) → Coolify install → claim admin
→ add server SSH key → deploy resource → Traefik FQDN
→ tunnel + Access (cloudflare-specialist) → healthz target
```
