# Process Management

Topic-wise reference on inspecting, controlling, and managing Linux processes.

## ps

Reports a snapshot of currently running processes (not live-updating).

Three option styles, which can technically be mixed but it's simpler to stick to one style at a time:
1. UNIX options — preceded by a dash
2. BSD options — no dash
3. GNU long options — preceded by two dashes

**Useful flags:**
- `-e` — every process (shows PID, TTY, TIME, CMD)
- `-ef` — full format, adds UID, PPID, C (%CPU), STIME
- `-eF` — extra full format, superset of `-f`, adds SZ (pages), RSS, PSR (last CPU core used)
- `-eo <cols>` — custom columns, e.g. `ps -eo pid,c,cmd`
  - `cmd` = full command string; `comm` = just the executable name (cleaner if you don't need args)
- `--sort <col>` — ascending by column; `--sort -<col>` — descending

**Go-to command:**
```bash
ps -eo pid,c,comm --sort -c | head -30
```

## htop

Interactive, real-time alternative to `ps`.
- Rule of thumb: `ps` for scripts, `htop` for actually investigating the system live.

**Key columns:**
- **RES** (Resident Set Size) — actual RAM the process is using
- **S** (state):
  - `R` — Running
  - `S` — Sleeping
  - `D` — Uninterruptible sleep
  - `Z` — Zombie (finished, but parent hasn't acknowledged/reaped it)
  - `T` — Stopped/paused
  - `I` — Idle kernel thread
- **MEM%** — RES as a percentage of total physical RAM
- **CPU%** — percentage of *one* core used right now; on an 8-core machine this can go up to 800%

## pgrep

Finds PIDs by process name.
- `pgrep brave` — list all matching PIDs
- `pgrep -c brave` — just the count
- `pgrep -l brave` — PID plus process name

## Zombie processes

A zombie is a finished process whose exit status hasn't been read by its parent yet — it lingers in the process table.

Script to detect them:
```bash
ZOMBIES=$(ps -eo pid,s,comm | awk '$2 ~ /^Z/ { print }')
```
Prints "No zombie processes found" if none exist.

## kill

- `kill <pid>` — sends a signal (default SIGTERM) asking the process to clean up (save state, close files) and exit gracefully. The process **can ignore it**. Always try this before force-killing.
- `kill -9 <pid>` — SIGKILL, handled directly by the kernel, not the process. The process is destroyed immediately with no chance to clean up — open files may not flush, temp files won't be removed. Last resort only.
- `killall <name>` — targets by process name instead of PID, e.g. `killall brave` or `killall -9 brave`. Useful when a program spawns many child processes and killing them all individually by PID would be tedious.

## Ctrl+C vs Ctrl+Z

Used to think these did the same thing. They don't:
- **Ctrl+C** sends **SIGINT**: asks the process to terminate (it can catch or ignore it).
- **Ctrl+Z** sends **SIGTSTP** ("terminal stop"): suspends the process. It stays frozen in memory, not killed, and is resumed with `fg` or `bg`.

## Background jobs

- `command &` — runs a command in the background immediately, e.g. `sleep 100 &`.
- `jobs` — lists background jobs in the current shell session.
  - The bracketed number is the job number.
  - `+` marks the current job (default target for `fg`/`bg`).
  - `-` marks the previous job.
  - States: Running, Stopped, Done.

## Scripts I've written (reference)

- **zombieProcesses** — reports any zombie processes' PID, state, and command (see awk snippet above).
- **processSnapshot** — continuously updating, RAM-sorted process list; flags anything over 500MB using `awk`. (See `shell-scripting.md` for the `$LINES` vs `$MAX_LINES` gotcha encountered while building this.)

## Init scripts without systemd

In containers with no init system, a daemon gets a SysV-style script using `start-stop-daemon` to background it and track it through a pidfile.

```sh
start-stop-daemon --start --background --make-pidfile --pidfile /var/run/foo.pid \
  --exec /usr/local/bin/foo -- --flag
start-stop-daemon --stop --pidfile /var/run/foo.pid
```
Lessons:
- **`--make-pidfile`** is for daemons that don't write their own. If the program manages its own pidfile (Grafana does), drop it. `--make-pidfile` writes the file as root *before* `--chuid` drops privileges, so the program can't overwrite it later.
- `--chuid <user>` runs the daemon as a non-root user. Anything it needs (pidfile directory, runtime dir, config) must be **owned** by that user, not merely mode-readable.
- Without systemd or apt postinst scripts, runtime directories (`/run/<svc>`) must be created explicitly.
- To translate a systemd unit into an init script, read the unit file and map `ExecStart`, `EnvironmentFile`, `User`/`Group`, `RuntimeDirectory`.
- When a daemon won't start via the script, **run the binary directly in the foreground** so errors print to the terminal.
- CRLF line endings in a script break the shebang and show up as a misleading "No such file or directory" (`git.md`).
- PID 1 in a container: if it exits, the container stops; use `reload`, not `restart`.

See `monitoring/prometheus-grafana.md` for where this was used.

