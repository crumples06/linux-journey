# Reverse Proxies & Caddy

## What a reverse proxy is

A server that sits between users and the actual backend server(s), so requests never hit the real server directly — they're all routed through the proxy first, which forwards them appropriately.

**Benefits:**
- Security — hides the real server's IP
- Performance
- Load balancing across multiple backend instances
- Handles traffic spikes more gracefully

**Common tools:** Nginx, HAProxy, Traefik, Apache, Caddy.

## Caddy

A web server I picked up specifically to front my self-hosted Gitea instance. Its standout feature: **automatic TLS/HTTPS** for everything it serves — this is what let me stop exposing Gitea directly on a bare port.

**Running it:**
- `caddy run` — starts in the foreground (terminal).
- `caddy start` — starts in the background.

**Configuration:**
- **Caddyfile** — simple, human-friendly config format.
- **JSON** — more powerful, machine-friendly.
- Caddy exposes an HTTP admin API (default `localhost:2019`) for changing config **without restarting**:
  ```bash
  curl localhost:2019/load -X POST -H "Content-Type: application/json" --data @caddy.json
  ```

## Reverse proxy in front of a Dockerized app (Gitea example)

**Core concept:** Caddy handles HTTPS and sits at the "front door"; the actual app just does its normal job on its own internal port, never exposed to the internet.

Request flow: `browser → Caddy (443, HTTPS) → app (e.g. port 3000, internal only)`

**Ports:**
- **80 / 443** are Caddy's ports, not the app's. Port 80 specifically matters for the **ACME HTTP-01 challenge** — how Let's Encrypt verifies domain ownership before issuing a cert.
- The app's own port (e.g. Gitea on 3000) never needs to be exposed publicly once Caddy is in front — Caddy reaches it internally over the Docker network using the service name:
  ```
  reverse_proxy gitea:3000
  ```

For how Caddy/Let's Encrypt actually confirms I "own" the domain (it's a DNS thing, not a Caddy thing), see `dns.md`.

## Trust & local-cert issues encountered

While testing locally before pointing a real domain at the server, ran into: `localhost` vs `.local` mDNS names vs raw LAN IP with a self-signed cert — none of these scale to letting *other people* connect without warnings or manual trust steps. Full breakdown of why (and the actual fix — a real domain + DuckDNS + public server) is in `dns.md`, since the root cause is entirely about how DNS and certificate trust work, not Caddy-specific config.
