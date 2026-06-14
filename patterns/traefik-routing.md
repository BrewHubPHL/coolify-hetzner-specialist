# Traefik Routing

**Impact:** HIGH  
**Tags:** traefik, domains, ssl, proxy

Coolify defaults to **Traefik** as reverse proxy.

## FQDN assignment

In app/service settings:

- Set FQDN (`brain.example.com`)
- Map container port → Traefik service port
- Ensure app listens on `0.0.0.0`, not only `127.0.0.1`

## Traefik + Cloudflare Tunnel (common sovereign stack)

| Exposure | Pattern |
|----------|---------|
| Public edge TLS | Cloudflare terminates TLS |
| Origin | Traefik on `https://localhost:443` |
| Tunnel route | **Origin Server Name** = public hostname; **No TLS Verify** ON |

Pointing tunnel at `:80` when Traefik forces HTTPS → redirect loop.

## Websocket / realtime

- Route to plain HTTP port (e.g. `:6001`)
- **No Access app** on websocket hostname
- App env: `PUSHER_HOST`, `PUSHER_PORT=443`

Verify: DevTools → Network → Socket → **101 Switching Protocols**

## SSL on Traefik directly

If not using Cloudflare edge:

- Let's Encrypt HTTP-01 needs ports 80/443 reachable **or**
- DNS challenge for wildcard certs

## Health checks & 502/503

| Error | Check |
|-------|-------|
| 502 Bad Gateway | Container port mismatch, crashed container |
| 503 No available server | Failing health check, wrong domain |
| 504 Gateway timeout | Slow upstream; increase proxy timeout |

## Basic auth

Traefik basic auth middleware for staging — **not** a substitute for Cloudflare Access on production admin UIs.

## References

- [Traefik overview (Coolify)](https://coolify.io/docs/knowledge-base/proxy/traefik/overview)
- [Domains](https://coolify.io/docs/knowledge-base/domains)
- `cloudflare-specialist` → `zero-trust-tunnels.md`
