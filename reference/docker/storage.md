# Docker Storage

Persisting and sharing data between containers and the host.

## Volumes

Persistent data stores **created and managed by Docker**, stored within a directory on the Docker host.

- Not a good choice if I need to access the files directly from the host — a volume is entirely managed by Docker, not meant for manual host-side access.
- Generally **better than writing directly into a container's writable layer**:
  - Doesn't increase the size of the container using it.
  - Faster — writing into a container's own writable layer requires a storage driver to manage that filesystem, which volumes bypass.

**Syntax:** `-v <volume_name>:<path_inside_container>`

Example — SQL Server data volume:
```bash
docker run -e MSSQL_SA_PASSWORD=pass -v mssql_data:/var/opt/mssql ...
```
If `mssql_data` doesn't already exist, Docker creates it automatically.

## Bind mounts

Mounts a specific file or directory **from the host machine** directly into the container — the opposite of a volume in that the host, not Docker, owns and manages the actual storage location.

**Good use cases:**
- Sharing source code or build artifacts between a dev environment on the host and a running container (e.g. live-reload dev setups).
- Generating files inside a container that need to persist to the host's filesystem afterward.
- Sharing configuration files from the host into one or more containers.

## Volumes vs bind mounts — when to use which

| | Volumes | Bind mounts |
|---|---|---|
| Managed by | Docker | Host filesystem directly |
| Host-side access to files | Awkward / not the point | Direct and easy |
| Performance | Generally faster (bypasses storage driver) | Depends on host filesystem |
| Typical use | App data that should persist but doesn't need manual editing (databases, etc.) | Source code, config files, dev workflows |

## docker cp (ad-hoc file transfer)

Not persistent storage, but related — `docker cp` copies files between the host and a running container one-off, e.g. moving a file from a Windows host into a container without setting up a mount at all.
