# Doppler Secrets at Container Boot

**Impact:** CRITICAL  
**Tags:** doppler, secrets, entrypoint

## Pattern: DOPPLER_WRAP

Wrap process startup so secrets never sit in Coolify UI plaintext for production tiers (or use Coolify secrets + Doppler sync operator-side).

```bash
#!/bin/bash
# entrypoint.sh (illustrative)
set -euo pipefail

run_app() {
  if [[ "${DOPPLER_WRAP:-}" == "1" && -n "${DOPPLER_TOKEN:-}" ]]; then
    exec doppler run -- "$@"
  else
    exec "$@"
  fi
}

if [[ $# -gt 0 ]]; then
  run_app "$@"
elif [[ -n "${APP_COMMAND:-}" ]]; then
  # shellcheck disable=SC2086
  run_app $APP_COMMAND
else
  run_app gunicorn -k uvicorn.workers.UvicornWorker "${APP_MODULE:-main:app}"
fi
```

## APP_COMMAND worker clone

Clone the same Docker image as the web app; change only env:

```
APP_COMMAND=python -m workers.ai_job_queue_drainer
```

Precedence: explicit docker CMD args > `APP_COMMAND` > default gunicorn.

## Doppler config map (illustrative)

| Runtime | Doppler config |
|---------|----------------|
| Python on Coolify | `brewhub/prd_python` |
| Next.js (if on Coolify) | project-specific — prefer edge Worker for customer app |

**Disjoint configs** — Python secrets don't belong in Netlify projection config.

## Coolify env UI trap

Don't paste production bridge URLs or override Doppler-managed keys in UI unless intentional — UI values can shadow entrypoint expectations.

## Rotation

Edit Doppler → restart Coolify resource (rolling restart). No `.env` in git.

## References

- BrewHub `docs/secrets-architecture.md` (in main monorepo)
- [Environment variables (Coolify)](https://coolify.io/docs/knowledge-base/environment-variables)
