# Shell Scripting

Topic-wise reference. Add new findings under the relevant section as I learn more, regardless of what week it happens.

## Script basics

- `set -euo pipefail` should go at the top of every script. It makes failures loud instead of silent — one wrong assumption cost me hours before I started doing this consistently.
  - Caught a real bug this way: a `shiftingParameters` script was using `$1` without checking it existed first, which used to fail silently.
- File permissions: any new script needs `chmod` to become executable. Considered changing the default via `umask` so I wouldn't have to, but decided it's not worth it — `chmod` is quick enough per-script.
- To run a script like a normal command (e.g. `helloworld` instead of `./helloworld.txt`):
  - The file must be on the `PATH` (check with `echo $PATH`) — I use `~/.local/bin`.
  - The filename must exactly match the command name, **no extension** (`helloworld`, not `helloworld.txt`).

## Variables

- Declare with `name="value"` — **no spaces around `=`**. `title = "System Information"` fails because `title` gets parsed as a command, not a variable assignment.
- Reference with `$title`.
- Environment variables are available the same way, e.g. `$HOSTNAME`.
- Watch out for shell "special" variables that look like normal names — `$LINES` is auto-populated with terminal row count, which silently broke a `head -n "$LINES"` constraint I'd written. Fixed by renaming my variable to `$MAX_LINES`.

## Output & redirection

- `command > file` redirects stdout to a file, e.g. `./sysinfo > ./sysinfo.html` to turn a script's HTML output into a real file.
- `>` overwrites; `>>` appends.
- `|` pipes stdout of one command into another.

## Exit status

- Every command returns an exit status on termination: `0` = success, anything else = failure.
- Check the previous command's status with `echo $?`.
- `true` → 0, `false` → 1. Reinforces that in Linux, 0 means success/true.

## test / conditionals

- `test` (or the `[ ]` shorthand) evaluates conditions, almost always paired with `if`.
- Space around `[` and `]` is mandatory.

```bash
if [ -f .bash_profile ]; then
    echo "You have a bash_profile"
else
    echo "You don't have a bash profile"
fi
```

## Positional parameters

- Command format: `command param1 param2 ...`
- Inside a script: `$0` = script name, `$1` = first param, `$2` = second, etc.
- `$#` = number of parameters passed (excludes `$0`).
- `shift` moves all positional parameters down by one (`$2` becomes `$1`, and so on) — useful for processing args in a loop.

### Command-line argument processing example

Added to a `sysinfo` script, handling three flags:
- `-i` — interactive mode, prompts for output filename
- `-h` — help text
- `-f` — accepts the output filename directly as an argument

## Flow control

```bash
# for loop
for i in {1..5}; do
    echo $i
done
```

```bash
# case
case word in
    pattern ) commands ;;
esac
```

```bash
# while
while [ condition ]; do
    commands
done
```

## trap

`trap` intercepts signals sent to a script and runs code in response instead of letting the script die immediately.

```bash
trap "echo 'Caught SIGINT'; exit" SIGINT
```

With this, Ctrl+C during the script prints the message first instead of just killing it.

## awk

Text-processing tool for structured data (logs, CSVs, command output). Processes input line by line.

```
awk [options] 'pattern {action}' input-file > output-file
```

- `$0` = whole line, `$1`/`$2`/... = individual fields (words), split by whitespace by default.
- `-v` passes an external shell variable into awk:
  ```bash
  awk -v threshold="$THRESHOLD" '$1>threshold {print $3}'
  ```

**Built-in variables:**
- `NR` — current record (line) number, starts at 1
- `NF` — number of fields in the current line (`$NF` = last field, handy)
- `FS` — field separator (default: whitespace; change for CSV etc.)
- `RS` — record separator
- `OFS` — output field separator
- `ORS` — output record separator

**Examples used in practice:**
```bash
ps -elf | awk '{print $1, $3, $5}'
```
```bash
awk 'NR==3, NR==6 {print NR,$0}' text.txt   # prints lines 3–6 inclusive
```
```bash
# find zombie processes: filter on state column starting with Z
ZOMBIES=$(ps -eo pid,s,comm | awk '$2 ~ /^Z/ { print }')
```
```bash
# flag processes over a RAM threshold in a live-updating script
ps ... | awk -v threshold="$THRESHOLD" '$mem_col > threshold { print }'
```

## Scripts I've written (reference)

- **sysinfo** — generates an HTML system-info page; has `-i`/`-h`/`-f` argument handling.
- **evenOdd** — checks if input is even/odd. Validates integer via regex, checks for 0, then uses modulo 2. Combines input reading, test expressions, exit codes, if statements, basic regex.
- **shiftingParameters** — practice with positional params; fixed an unbound-variable bug by checking `$1` exists before use.
- **zombieProcesses** — reports zombie process PID/state/command using `ps` + `awk` (see above).
- **processSnapshot** — continuously updating process list sorted by RAM usage, flags processes over 500MB using `awk`. Ran into the `$LINES` gotcha above while building this.
- **boot-time-check** — checks userspace boot time against a threshold, fires a `notify-send` alert if exceeded. Currently uses a **hardcoded threshold** — plan to replace with a rolling average later. Wired up as a systemd user service (see `systemd.md`).
