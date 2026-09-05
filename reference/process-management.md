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

Used to think these did the same thing — they don't:
- **Ctrl+C** (SIGINT) — terminates the process.
- **Ctrl+Z** (SIGSTOP) — pauses/suspends the process; it stays frozen in memory, not killed, and can be resumed with `fg` (foreground) or `bg` (background).

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
