# Dockerfile & Images

Building images: instructions, layers, and lessons from actual builds.

## What a Dockerfile is

A plain text file of instructions for building a Docker image. **Each instruction creates a new layer** in the image.

**Key components:**
- **Base image** — starting point (`FROM`).
- **Application code and dependencies** — added and installed via instructions.
- **Commands and configuration** — execute commands, set env vars, expose ports.

## Common instructions

| Instruction | Purpose |
|---|---|
| `FROM` | specifies the base image, e.g. `FROM python:3.9` |
| `COPY` | copies files from host into the image |
| `RUN` | executes a command during the **build** process |
| `CMD` | default command run when a container **starts** |
| `WORKDIR` | sets the working directory inside the container |
| `EXPOSE` | documents which port the container listens on |

## Image layers

- Each filesystem-modifying instruction = one layer, stacked on top of previous ones. Layers reuse functionality from the layers below them.
- **If a layer changes, every layer after it must be rebuilt.** This is why Dockerfiles are ordered with the least-frequently-changing instructions first (e.g. installing dependencies) and the most-frequently-changing ones last (e.g. copying application code) — it maximizes Docker's build cache reuse.
- Images can be built on top of other custom images, not just official base images — e.g. package up an application as a base image, then extend it with more functionality in a separate image without rebuilding the application each time.

## Example: processSnapshot script in a container

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache bash procps util-linux
WORKDIR /app
COPY processSnapshot /app/processSnapshot
RUN chmod +x /app/processSnapshot
CMD ["bash", "/app/processSnapshot"]
```

**Gotcha:** running this script inside a container normally only shows the **container's own PID namespace** — not the host's actual processes. For a monitoring script like this to see the host machine's processes, the container needs to share the host's PID namespace:
```bash
docker run --rm -it --pid=host process-snapshot
```
This weakens container isolation, so it's better suited to a container that's *actually doing other work* and monitoring its own processes, rather than as a general host-monitoring tool.

## Example: basic Python server

```dockerfile
FROM python
COPY server.py /server/
WORKDIR /server
CMD python3 server.py
```

**Build & run flow:**
```bash
docker build .                          # run from the directory containing Dockerfile + server.py
docker images -a                        # -a needed because an unnamed build shows as <none> otherwise
docker image tag <image-id> python-server:latest   # name it after the fact
docker run -it python-server            # -it = interactive with a terminal
docker run -it -p 8080:8080 python-server   # expose port to actually reach the server
curl http://localhost:8080
```

**Things I tripped on:**
- Forgetting to name the image at build time left it as `<none>` in `docker images` (had to use `-a` to even see it, then tag it after the fact).
- The container's internal port isn't reachable from the host until explicitly published with `-p host_port:container_port`.
