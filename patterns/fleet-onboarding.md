# Fleet Onboarding Runbook

**Impact:** CRITICAL  
**Tags:** onboarding, tunnel, tailscale, healthz

Abstract runbook for adding a **new fleet machine** (shop box, Hetzner brain, Pi). Pair with `cloudflare-specialist` `zero-trust-tunnels.md`.

## Sequence (order matters)

```
1. OS hardening
2. Tailscale + SSH
3. Coolify install + claim admin
4. ONE cloudflared tunnel (named after machine)
5. Deploy services + Traefik FQDNs
6. Access apps (dashboard BEFORE dependent deploys)
7. Healthz monitor targets
8. Verify no public inbound exposure
```

## Step detail

### 1. OS hardening

- Ubuntu LTS (or fleet standard)
- Full disk encryption where available
- `tailscale up --ssh`
- Ubuntu Pro / livepatch if applicable

### 2. Coolify

- Install via official script
- Claim admin **immediately**
- Register server in Coolify with SSH key

### 3. Tunnel

- **One tunnel per machine** — routes added inside as services appear
- `cloudflared` as **systemd** (dashboard connector command)
- Deleting tunnel may leave stale DNS CNAMEs — clean zone before re-add

### 4. Access policies

| Audience | Policy |
|----------|--------|
| Humans | Email allow-list |
| Monitors / Workers | Service Auth on path-scoped app (e.g. `/api/health`) |
| Websocket hosts | No Access — use app-level auth |

**Most specific path wins** between overlapping apps.

### 5. Healthz

Before merging code that probes a new URL:

1. Create Access Service Auth app for health path
2. Add monitor target with `redirect: 'manual'` (302 = DOWN)
3. Deploy monitor change

### 6. Exposure verification

From cellular (not tailnet):

- Hit literal public IPv6:port for a published Docker port — should **fail**
- Confirm only tunnel hostnames resolve and pass Access

Docker **bypasses ufw** — never treat host firewall as sufficient.

## Post-onboarding

- Add Sentinel / disk alerts in Coolify
- Schedule DB backups to R2
- Document in private `brew-hub-overrides/` (hostname table)

## References

- [Cloudflare Tunnels (Coolify)](https://coolify.io/docs/integrations/cloudflare/tunnels)
- [Automated Docker cleanup](https://coolify.io/docs/knowledge-base/server/automated-docker-cleanup)
