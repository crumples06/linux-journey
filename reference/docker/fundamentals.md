# Docker Fundamentals

Core concepts and CLI basics.

## What Docker is

Containerizes applications so they run in a consistent environment regardless of the host machine.

**Architecture:** client-server model. The **Docker client** talks to the **Docker Daemon**, which builds, runs, and manages containers. They communicate over a REST API, via Unix sockets or a network interface.

**Components:**
- **Docker Engine** — handles creation and management of containers.
- **Dockerfile** — instructions for building an image (see `dockerfile.md`).
- **Docker Image** — template used to create containers; contains app code + dependencies.
- **Docker Hub** — cloud repository for finding/sharing images.
- **Docker Registry** — general storage/distribution system for images.

## Core CLI commands

| Command | What it does |
|---|---|
| `docker run` | create and start a container from an image |
| `docker ps` | list running containers |
| `docker pull <image>` | download an image from a registry |
| `docker images` | list images present on this machine |
| `docker images -a` | list all images, including unnamed ones (`<none>`) |
| `docker rmi <image>` | remove an image |
| `docker logs <name\|id>` | view container logs |
| `docker logs -f <name\|id>` | follow logs in real time |
| `docker logs --tail 30 <name\|id>` | last 30 log lines only |
| `docker logs --since 5m <name\|id>` | logs from the last 5 minutes only |
| `docker exec <name\|id> <cmd>` | run a command inside a running container |
| `docker stop <name\|id>` | request graceful stop (grace period, then force-kill if needed) |
| `docker kill <name\|id>` | force-close immediately, no grace period |
| `docker inspect <name\|id>` | low-level JSON info on a container/image |
| `docker cp` | copy files between host and container |

Full reference for anything else: `docker --help` — there's a lot more than the above.

### Things I learned the hard way

- **`docker logs` only shows output from PID 1** — the main process the container was started with. Ran an `echo` via `docker exec` expecting to see it in `docker logs` and it never showed up. This is intentional: `exec` is meant for quick debugging and shouldn't clutter the primary process's logs.
- **Port conflicts are silent and confusing.** Spent hours debugging a Node app that couldn't connect to a containerized SQL Server — turned out the container and the *host's own* SQL Server install were both listening on the same port, so my app kept connecting to the host instance instead of the container. Lesson: **always check if a port is already in use on the host before mapping it.**
- **`docker inspect --format`** can extract specific fields, e.g.:
  ```bash
  docker inspect sql_dev --format "{{.Config.Env}}"
  ```
  There's a lot you can format this way, but for casual inspection it's often just easier to look in Docker Desktop.
- **Quoting passwords with special characters (like `$`) is shell-dependent, not Docker-dependent:**
  - PowerShell needs single quotes.
  - `cmd.exe` needs double quotes (or none).
  - Tools like `dotenvx` do their own `$`-expansion inside `.env` files — different rules again.
- **`docker cp`** — used this to copy files between the Windows host and a container directly.

## Environment variables in containers

`-e` sets environment variables at `docker run`/`docker exec` time, e.g.:
```bash
docker run -e MSSQL_SA_PASSWORD=pass ...
```
Used to set the SQL Server SA (system administrator) login password on container start.

## Running commands inside a container

`docker exec` runs commands against an already-running container. Example — running `sqlcmd` inside a SQL Server container:
```bash
docker exec -it sql_dev /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "YourPassword" -C -Q "SELECT 1"
```
- `/opt/...` — path to the `sqlcmd` binary **inside** the container.
- `-S localhost` — connect to localhost because we're already inside the container.
- `-C` — trust the server certificate without validation.
- `-Q` — the SQL query to run.

Docker Desktop also has a built-in terminal for running commands inside a container without typing `docker exec` each time.
