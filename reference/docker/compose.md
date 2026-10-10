# Docker Compose

Multi-container orchestration on a single host. Light so far — filling this in as the Gitea project grows.

## What I'm using it for

Self-hosted Gitea git server: the `docker-compose.yml` defines two services —
- **gitea** — the git server itself
- **mysql** — database backing storage for Gitea

Compose starts both together and (per `networking.md`) puts them on a shared network where they can address each other by service name rather than IP.

Service name vs container name
The top-level YAML key (api, frontend, sql_dev) is the service name. docker compose exec/logs/restart/ps use it, and so does internal DNS between containers.
container_name: is a cosmetic override of the Docker Engine-level name (docker ps, docker logs <name>). Compose subcommands do not accept it.
Docker's internal DNS resolves both names as aliases on the same network, but docker compose <cmd> wants the service name.
Environment variables
A root .env (next to docker-compose.yml) is used by Compose itself for ${VAR} substitution inside the compose file.
A service only receives a variable if it is listed under that service's environment: (or provided via env_file:).
Compose-injected variables exist before the app starts, so dotenv.config() in Node becomes a no-op for anything Compose already set.
environment: = runtime values read by the running process. build.args = values injected during image build, matched to an ARG in the Dockerfile.
ports vs expose
ports: "80:80" publishes a port to the host (reachable from outside Docker).
expose: ["80"] only makes it reachable by other containers on the same network. Nothing is published to the host.
Healthchecks and --wait
yaml
healthcheck:
  test: ["CMD-SHELL", "command -v ansible >/dev/null && command -v curl >/dev/null"]
  interval: 5s
  retries: 10
CMD does not use a shell, so a single string with spaces fails as a literal binary name. Use CMD-SHELL for &&, pipes and multiple commands.
docker compose up --build --wait blocks until services are healthy. Useful in CI (ci-cd/github-actions.md).
Day-to-day commands
bash
docker compose up -d
docker compose down
docker compose logs -f [service]        # watch requests ripple through the stack
docker compose restart <service>
docker compose exec <service> printenv | grep <VAR>    # what env vars does it REALLY have?
docker compose build --no-cache <service>
docker compose up -d --force-recreate <service>

When in doubt, rebuild forcibly. A lot of my problems came from stale Dockerfiles or .env files.

Still to cover here
depends_on and startup ordering (e.g. making sure mysql is ready before gitea connects; healthcheck-based conditions)
Named volumes and bind mounts at the Compose level vs CLI flags
docker-compose.yml syntax in depth (volumes block)
How this connects to Gitea Actions / CI once I reach that part
