# Linux / Bash Reference (compiled from Week 1–5 notes)

---

## 1. Filesystem Navigation

- `cd -` → last working directory
- `cd /` → filesystem root (above home)
- `cd ~` → home directory (different from `cd /`)

**`ls`**
- `-l` : long format → `type + permissions | links | owner | group | size | mtime | name`
  - first char: `-` regular file, `d` directory
  - next 9 chars: owner / group / other permissions (rwx each)
  - `--human-readable` → sizes in KB/MB/GB instead of raw bytes
- `--all` : show hidden files (`.git`, `.gitignore`, etc.)
- `--classify` : appends `/` to dirs, `*` to executables
- Options can be chained: `ls -l --all --human-readable`
- Wildcards affect `ls` results directly — e.g. `ls l*` matches files/dirs starting with `l`, and if any matched item is a directory, its contents get listed too.

**`cat`**
- `cat file` → print to terminal
- `cat file1 file2 > file3` → merge into new file (overwrites file3)
- `cat file1 >> file2` → append, preserves existing content
- `cat > file.txt` → create file and type into it directly
- `--number` → show line numbers

**`less`**
- `less filename` → interactive, scrollable, searchable — better than `cat` for reading

**`cp`**
- `cp file1 file2` — same behavior as `cat file1 > file2`
- `cp file1 dir1` — copy file directly into a directory
- `cp -R dir1 dir2` — recursive; creates `dir2/dir1/...` (nests, doesn't merge)

**`mv`**
- Rename: `mv file1 file2`
- Move: `mv file1 dir1`
- `mv dir1 dir2` — renames if dir2 doesn't exist, moves dir1 into dir2 if it does

**`rm`**
- `rm file1 file2`
- `rm -r dir1 dir2`

**Wildcards**
- `*` all names, `g*` starts with g, `d??` starts with d + exactly 2 more chars
- `[abc]` one char matching a, b, or c → `[abc]*` combines with prefix matching
- POSIX classes: `[[:upper:]]`, `[[:alnum:]]`, `[[:alpha:]]`, `[[:digit:]]`, `[[:lower:]]`
- Negate with `[!...]`, e.g. `[![:lower:]]`

---

## 2. Permissions

- `chmod` sets rwx for owner/group/other, each representable as 3 bits (binary) → one octal digit
- e.g. `755` = `111 101 101` = `rwxr-xr-x`
- `chmod 755 file`

---

## 3. I/O Redirection & Pipes

- `>` redirect stdout to a file (overwrites)
- `>>` append stdout to a file
- `|` pipe — feeds stdout of one command into stdin of the next (e.g. `ls -l | less`)
- Same `>` logic applies to running scripts: `./sysinfo > sysinfo.html`

---

## 4. Shell Basics

**Variables**
- `title="System Information"` — **no spaces** around `=`, or the shell tries to run it as a command
- Access with `$title`
- Environment variables work the same way, e.g. `$HOSTNAME`, `$PATH`, `$LINES` (terminal row count — watch out, it can silently override a variable you meant to use for line limits)

**Arithmetic**
- `echo $((2+3+7))`
- `echo $(((5**2)+3))`

**Brace expansion**
- `echo F-{A,B,C}-B` → `F-A-B F-B-B F-C-B`
- `echo {1..5}` → `1 2 3 4 5`

**Aliases & functions**
- `alias l='ls -l'` — add to `~/.bashrc` to persist, or run directly for session-only
- Multi-line logic → shell functions:
  ```bash
  today() {
    echo -n "Today's date is "
    date +"%A, %B %-d, %Y"
  }
  ```

**Running your own scripts as commands**
- Shell searches directories listed in `$PATH` (in order) whenever you type a command
- Put a script in one of those dirs (e.g. `~/.local/bin`) to run it like a built-in command
- File must be named exactly as the command (no extension — `helloworld`, not `helloworld.txt`)

---

## 5. Scripting Fundamentals

**Exit status**
- Every command returns an exit status on termination: `0` = success, anything else = failure
- Check with `echo $?` right after the command
- Counterintuitive at first: `true` → 0, `false` → 1

**`test` / `[ ]`**
- `test expr` or `[ expr ]` (space around brackets is mandatory)
  ```bash
  if [ -f .bash_profile ]; then
    echo "You have a bash_profile"
  else
    echo "You don't have a bash profile"
  fi
  ```

**Positional parameters**
- `$0` = script name, `$1`, `$2`, … = arguments
- `$#` = number of parameters (excludes `$0`)
- `shift` moves all params down by one (`$2`→`$1`, etc.)

**Flow control**
```bash
for i in {1..5}; do
    echo $i
done

case word in
    pattern ) commands ;;
esac

while [ condition ]; do
    commands
done
```

**Command-line argument parsing pattern** (used in `sysinfo`)
```bash
while [ "$1" != "" ]; do
    case $1 in
        -f | --file )       shift
                             filename=$1
                             ;;
        -i | --interactive ) interactive=1 ;;
        -h | --help )        usage; exit ;;
        * )                  usage; exit 1
    esac
    shift
done
```

**`trap`** — intercept signals inside a script
- `trap "echo 'Caught SIGINT'; exit" SIGINT` → runs on Ctrl+C instead of dying silently

---

## 6. Process Management

**`ps`**
- Three option styles (UNIX `-`, BSD none, GNU `--`) — mixing across styles can conflict; stick to one family
- `-e` : every process (PID, TTY, TIME, CMD)
- `-ef` : full format, adds UID, PPID, %CPU, STIME
- `-eF` : extra-full, adds SZ, RSS, PSR (last CPU core used)
- `-eo <cols>` : custom columns, e.g. `ps -eo pid,c,cmd`; use `comm` instead of `cmd` for just the executable name
- `--sort c` / `--sort -c` : ascending/descending by CPU
- Useful one-liner: `ps -eo pid,c,comm --sort -c | head -30`

**`htop`**
- Real-time, interactive — better for live investigation; `ps` is better for scripts
- Columns: RES (RAM used), S (state: R running, S sleeping, D uninterruptable sleep, Z zombie, T stopped, I idle kernel thread), MEM%, CPU% (per-core; can exceed 100% on multi-core)

**`kill`**
- `kill pid` : graceful request (process can ignore it) — try this first
- `kill -9 pid` : kernel force-destroys the process; open files may not flush, temp files may persist — last resort
- `killall process_name` : kill by name, useful for multi-process apps (e.g. browsers)

**Jobs & signals**
- `&` at end of command → runs in background immediately (`sleep 100 &`)
- `jobs` → lists background jobs (`+` = current/default for fg/bg, `-` = previous, plus state)
- Ctrl+C (SIGINT) terminates; Ctrl+Z (SIGSTOP) suspends — resume with `fg`/`bg`, doesn't kill

**`pgrep`**
- `pgrep name` → PIDs of matching processes
- `pgrep -c name` → count only
- `pgrep -l name` → PID + process name

**Zombie processes**
- Detect with: `ps -eo pid,s,comm | awk '$2 ~ /^Z/ { print }'`
- Zombie = finished process whose parent hasn't reaped it yet (state `Z`)

---

## 7. `awk` (text processing)

- Syntax: `awk [options] 'pattern {action}' input-file`
- Processes line by line; `$1`, `$2`, … = fields, `$0` = whole line
- Built-ins:
  - `NR` — current record/line number (starts at 1)
  - `NF` — number of fields in current line (`$NF` = last field)
  - `FS` — input field separator (default: space)
  - `RS` / `OFS` / `ORS` — record/output field/output record separators
- `-v var=value` : inject an external shell variable into awk
  - `awk -v threshold="$THRESHOLD" '$1>threshold {print $3}'`
- Examples:
  - `ps -elf | awk '{print $1, $3, $5}'`
  - `awk 'NR==3, NR==6 {print NR,$0}' text.txt` → prints lines 3–6 inclusive

---

## 8. systemd & `systemctl`

- systemd = first process started by the kernel; manages user space init, services, and core OS tasks
- Everything is a **unit** — services, sockets, targets, timers, mounts, etc., each with a suffix (`.service`, `.socket`, …)
- Unit relationships:
  - `Wants=` / `Requires=` — dependency strength
  - `After=` / `Before=` — ordering only, no implied dependency
  - `Conflicts=` — mutual exclusion

**Inspecting services**
- `systemctl status <service>` — full status: load state, active state, main PID, cgroup, recent logs
  - `disabled` = not set to start at boot (may still be running if started another way)
  - cgroups group related processes, which is why the group name alone (`gdm`) works without needing the specific process name (`gdm3`)
- `systemctl is-active <service>` / `is-failed <service>` — same output, active or not
- `systemctl is-enabled <service>` — one-word enabled/disabled status
- `systemctl list-units` — all loaded units (huge, rarely useful directly)
- `systemctl list-units --type=service` — narrow to services
- `systemctl list-units --type=service --state=running` (or `--state=failed`) — narrow further
- `systemctl list-dependencies <target>` — check what depends on a unit before disabling/masking it

**Controlling services**
- `sudo systemctl start/stop/restart <service>`
- `restart` = stop then start
- `reload` = re-read config without full interruption (not all services support it)
- `sudo systemctl mask <service>` — fully disable, prevents it from starting even as a dependency

**`journalctl`** (integrates with systemd's journald)
- `journalctl -u <service>` — logs for a specific service
- `journalctl -fu <service>` — follow logs in real time
- `journalctl -u <service> -b` — logs since last boot
- `journalctl -u <service> --since "YYYY-MM-DD HH:MM:SS" --until "YYYY-MM-DD HH:MM:SS"` — logs in a specific window

---

## 9. Boot Time Analysis — `systemd-analyze`

- `systemd-analyze blame` — lists units from last boot with time taken (starting → active), sorted slowest first
- `systemd-analyze critical-chain` — shows the critical path that determined total boot time
- Real example from notes: `NetworkManager-wait-online.service` was blocking boot for 5.485s waiting for network
  - Confirmed nothing depended on it via `systemctl list-dependencies network-online.target`
  - Masked it: `sudo systemctl mask NetworkManager-wait-online.service`
  - Result: boot time dropped from 10.538s → 8.415s (2.1s faster)

---

## Quick-reference command index
| Task | Command |
|---|---|
| Real-time process monitor | `htop` |
| Sorted process snapshot | `ps -eo pid,c,comm --sort -c \| head -30` |
| Force kill | `kill -9 <pid>` |
| Kill by name | `killall <name>` |
| Find zombies | `ps -eo pid,s,comm \| awk '$2 ~ /^Z/'` |
| Service status | `systemctl status <svc>` |
| Service logs (live) | `journalctl -fu <svc>` |
| Boot time breakdown | `systemd-analyze blame` |
| Disable a slow unit | `sudo systemctl mask <svc>` |
