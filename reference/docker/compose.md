# Docker Compose

Multi-container orchestration on a single host. Light so far — filling this in as the Gitea project grows.

## What I'm using it for

Self-hosted Gitea git server: the `docker-compose.yml` defines two services —
- **gitea** — the git server itself
- **mysql** — database backing storage for Gitea

Compose starts both together and (per `networking.md`) puts them on a shared network where they can address each other by service name rather than IP.

## Still to cover here

This file is intentionally thin right now — Compose specifics I haven't dug into yet but expect to need soon:
- `docker-compose.yml` syntax in depth (services, networks, volumes blocks)
- Environment variable management across services (`.env` files, `environment:` vs `env_file:`)
- `depends_on` and startup ordering between services (e.g. making sure `mysql` is ready before `gitea` tries to connect)
- Named volumes and bind mounts defined at the Compose level vs passed via CLI flags
- `docker compose up -d`, `down`, `logs`, `restart` workflows day-to-day
- How this connects to Gitea Actions / CI-CD once I get to that part of the roadmap
