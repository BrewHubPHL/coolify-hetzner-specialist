# Anti-Patterns

## Coolify

### ❌ Leaving instance unclaimed after install
Scanners register open Coolify panels.

### ❌ Wrong container port in FQDN mapping
502 loop — app listens on different port than Traefik expects.

### ❌ App binds 127.0.0.1 only inside container
Traefik can't reach it — use `0.0.0.0`.

### ❌ Changing Supabase pooler tenant ID after first deploy
Requires DB migrations — set pre-boot.

### ❌ Overriding Doppler-managed secrets in Coolify UI accidentally
Shadows entrypoint contract; causes prod/staging drift.

## Hetzner / network

### ❌ Relying on ufw to block Docker-published ports
Docker iptables rules bypass ufw.

### ❌ Skipping snapshot before major cutover
No fast rollback.

### ❌ Public `:8000` Coolify dashboard without Access
Admin plane exposed.

## Traefik / tunnels

### ❌ Tunnel route to Traefik `:80` with HTTPS redirect enabled
Infinite redirect — use `https://localhost:443` + origin SNI.

### ❌ Access policy on websocket hostname
Handshake can't complete login redirect.

### ❌ Deploy monitor before Access Service Auth exists
False outage pages.

## Operations

### ❌ One tunnel per hostname
Orphan DOWN tunnels + stale DNS.

### ❌ Backup configured but never restore-tested
Discover corrupt backups during incident.

### ❌ Building heavy images on tiny VPS without build server
OOM during deploy — use external build server or GitHub Actions.

## Agents

### ❌ Quoting Coolify install script from memory
Fetch current docs — URL/script changes.
