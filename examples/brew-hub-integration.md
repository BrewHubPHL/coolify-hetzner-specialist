# BrewHub Integration (Abstract)

## Fleet roles (generic)

| Box role | Typical workloads |
|----------|---------------------|
| **Brain box** (Hetzner) | Python ADK, franklin MCP, cron workers, queue drainer |
| **Shop box** | Coolify for shop services, optional self-hosted Supabase |
| **Pi box** | Edge hardware services (scale, sensors) |

All exposed via **one tunnel per machine** + Cloudflare Access — not public SSH or open Docker ports.

## Python tier on Coolify

```
Git push → Coolify build (Dockerfile)
→ entrypoint.sh (DOPPLER_WRAP=1)
→ doppler run → gunicorn/uvicorn (brain, MCP)
```

Worker clone: duplicate app, set `APP_COMMAND` to drainer module — same image, different process.

Customer chat **does not** require MCP bridge if loyalty reads Supabase-direct — keeps customer path off single brain box.

## Background crons

Marketing/weather/sports crons as **Coolify Scheduled Tasks** dispatching `APP_COMMAND` — not customer-facing, write to Postgres + email alerts.

## Healthz chain

```
Worker monitor → Service token → path-scoped Access
→ GET /api/health → 200 or page SMS
```

302 to Access login = DOWN (`redirect: 'manual'`).

## Customer vs staff surfaces

| Surface | Hosting choice |
|---------|----------------|
| Customer web/mobile | Next.js on Cloudflare Workers (OpenNext) |
| Staff MCP / brain | Coolify on Hetzner behind Access |
| Money API | Dedicated API Worker |

Coolify is for **long-running Python**, self-hosted data plane, and shop ops — not the customer checkout hot path.

## Kill switches

- No production brain cutover without Hetzner snapshot
- No "temporary" public Coolify dashboard
- No basic-auth-only on services that should use Access

## Private overrides

Tunnel names, server IPs, SSH key paths → private overlay repo.
