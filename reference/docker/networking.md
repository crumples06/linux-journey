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

## Custom networks vs the default bridge

- Containers on the same custom `networks:` entry can reach each other by service name as a hostname (Docker's embedded DNS).
- The default bridge network does not provide that name resolution, which is why a custom network is needed.
- Isolation by network: put each container only on the networks it needs. In the bastion project the bastion sits on `public` and `private`, the servers only on `private`, the CA alone on `ca`. A container on one network cannot reach one that is only on another. See `projects/bastion-ssh-ca.md`.
- Publish an API to the host loopback only when other host processes need it: `127.0.0.1:9000:9000`.

Proving DNS and connectivity from inside a container
```bash
getent hosts <service>        # proves name resolution works
docker network ls
docker network inspect <name> # which containers are on it, and their IPs
```
- Resolving a name does not prove a connection works. Always follow up with a real query or request.

`ports` vs `expose`
- `ports: "8080:80"` publishes to the host.
- `expose: ["80"]` is container-to-container only. (Details in `compose.md`.)

## Docker socket warning

Mounting `/var/run/docker.sock` into a container is root-equivalent on the host and bypasses every network boundary. Prefer calling a service's network API (the CA's HTTPS API, for example) over driving containers with `docker exec` from inside another container.

## Still to cover here

- `--network` host and when to use it
- How Compose actually wires up the shared network between services (see `compose.md`)
- `--pid=host` and other namespace-sharing flags (touched on briefly in `dockerfile.md` for the `processSnapshot` container) — worth a deeper look at namespaces generally
