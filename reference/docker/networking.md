# Docker Networking

Ports, port publishing, and reaching containers from outside and from each other.

## Publishing ports

A container's internal port is **not reachable from the host** until explicitly published:
```bash
docker run -it -p 8080:8080 python-server
```
Format is `-p <host_port>:<container_port>`. Without `-p`, the app inside the container can be running fine but nothing outside the container can reach it.

## Port conflicts (host vs container)

**The lesson that cost me the most debugging time so far:** ran a SQL Server container and a Node app that needed to connect to it. The container worked fine when queried directly from a terminal, but the Node app couldn't connect — no matter what I changed.

Root cause: the SQL Server **inside the container** and the SQL Server **installed on the host machine** were both listening on the same port. My Node app was silently connecting to the host's SQL Server instance instead of the container's, because the host service got priority on that port.

**Takeaway:** always check whether a port is already in use on the host *before* mapping a container to it. This class of bug is silent — nothing errors out, it just connects to the wrong thing.

## Container-to-container networking (via service name)

When containers are on the same Docker network (e.g. defined together in a `docker-compose.yml`), they can reach each other **by service name** instead of an IP or `localhost` — Docker's internal DNS resolves it.

Example from my Gitea + Caddy setup: Caddy reverse-proxies to Gitea using the service name, not an IP or port exposed to the host:
```
reverse_proxy gitea:3000
```
This means Gitea's own port (3000) never needs to be exposed to the internet at all — Caddy reaches it purely over the internal Docker network. See `networking/reverse-proxy.md` for the full reverse-proxy writeup; this is the Docker-networking half of that same setup.

## Still to cover here

- Docker's default bridge network vs custom networks vs `--network host`
- How Compose actually wires up the shared network between services (see `compose.md`)
- `--pid=host` and other namespace-sharing flags (touched on briefly in `dockerfile.md` for the `processSnapshot` container) — worth a deeper look at namespaces generally
