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

## Containers without an init system

When a container runs just `sshd -D` (or any single daemon) as PID 1, nothing supervises children:
- **If PID 1 exits, the container stops.** `service ssh restart` kills the container, because the process Docker is watching ends. Use `reload` to re-read config in place.
- `systemd`-based tooling (the Ansible `systemd` module, `systemctl`) fails. Use `service` or the init script directly.
- Things systemd and apt postinst scripts do invisibly (create `/run/<svc>`, plugin directories, pidfile ownership) have to be done explicitly.
- Service names follow distro convention: on Ubuntu the init script is `ssh`, though the binary is `sshd`. Check `ls /etc/init.d/`.
- Non-interactive `apt-get install` needs `-y`, or the build/task hangs waiting for confirmation.

## `exec` and signal handling in entrypoints

```sh
#!/bin/sh
ssh-keygen -A          # generate host keys that don't exist yet
exec sshd -D           # replace the shell with sshd
```
- Without `exec`, the shell script stays PID 1 and `docker stop`'s signal never reaches `sshd`, so Docker waits out its timeout and then kills it.

## Don't bake secrets or identities into an image layer

- Packages like `openssh-server` generate **host keys at install time**. Containers from one image would share identical host keys, defeating the point of host identity.
- Fix: delete them in the **same `RUN`** as the install (so they never persist in a layer), and generate them at container start in the entrypoint. Store them on a named volume so they survive recreation.

## Build-time vs runtime reminders

- `ARG` + `build.args`: build time. `ENV`/`environment:`: runtime.
- `RUN` runs at build, `CMD` at container start.
- `CMD` in a **healthcheck** `test:` does not use a shell, `CMD-SHELL` does. See `compose.md`.

## Changing what the images contain for Ansible targets

For Ansible to manage a plain container, the image needs Python and a sudo-capable user:
```dockerfile
RUN apt-get update && apt-get install -y openssh-server sudo python3
RUN echo "admin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/admin && chmod 440 /etc/sudoers.d/admin
```
(Passwordless sudo is a deliberate lab tradeoff; see `security/access-control.md`.)
