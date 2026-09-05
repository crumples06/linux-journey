# systemd

Topic-wise reference on systemd, units, and boot analysis.

## What systemd is

The system and service manager for Linux — the first process the kernel starts, responsible for initializing user space, managing services, and handling core OS tasks.

Everything in systemd is a **unit** — the fundamental building block. Every service, socket, device, mount point, timer, or group of units is represented as a unit, distinguished by its filename suffix (`.service`, `.socket`, `.target`, `.timer`, `.mount`, etc).

**Unit relationships:**
- `Wants=` / `Requires=` — dependency strength (soft vs hard dependency)
- `After=` / `Before=` — ordering only, doesn't imply a dependency
- `Conflicts=` — prevents two units running simultaneously

## systemctl

Utility for controlling systemd.

### Checking status

`systemctl status <service>` gives detailed info, e.g. `systemctl status gdm`:
```
● gdm.service - GNOME Display Manager
     Loaded: loaded (...; disabled; preset: enabled)
     Active: active (running) since ...
    Process: ... ExecStartPre=... (code=exited, status=0/SUCCESS)
   Main PID: 1930 (gdm3)
      Tasks: 5 (limit: 22931)
     Memory: 6.6M
     CGroup: /system.slice/gdm.service
             └─1930 /usr/sbin/gdm3
```

Notes on reading this output:
- `disabled` means the service isn't configured to autostart at boot — if I see this on a running service, something else started it.
- **Main PID** is the service's primary process.
- **Tasks** = number of threads/processes in the service's **cgroup** (control group), including children. This is why commands can target a group name (`gdm`) rather than the exact process name (`gdm3`) — cgroups organize processes hierarchically under the unit.
- The dated lines at the bottom are the service's recent log entries.

**Quick status checks:**
- `systemctl is-active <service>` / `systemctl is-failed <service>` — same output, tells you if it's currently active
- `systemctl is-enabled <service>` — one-word answer: enabled or disabled

### Listing units

- `systemctl list-units` — lists all loaded units (huge, not very usable)
- `systemctl list-units --type=service` — filter by type
- `systemctl list-units --type=service --state=running` — filter by type + state (`--state=failed` for failures)

### Controlling services

- `sudo systemctl start <service>` / `stop` / `restart`
- `restart` — stops then starts
- `reload` — tells the service to re-read its config without a full restart (not all services support this)
- `sudo systemctl mask <service>` — prevents a unit from being started at all (used this on `NetworkManager-wait-online.service`, see below)

### Checking dependencies

`systemctl list-dependencies <target>` — e.g. used `systemctl list-dependencies network-online.target` to confirm nothing depended on a service before masking it.

## journalctl (systemd's logging)

- `journalctl -u <service>` — logs for a specific service (can go back a long way, hard to jump to a specific date manually)
- `journalctl -fu <service>` — follow logs live for that service
- `journalctl -u <service> -b` — logs since the last boot
- `journalctl -u <service> --since "2026-07-13 10:00:00" --until "2026-07-13 11:00:00"` — logs within a specific time window (no need for a custom script — this flag combo does it directly)

## systemd-analyze (boot performance)

- `systemd-analyze blame` — lists every unit involved in the last boot along with how long each took to go from *starting* to *active*.
- `systemd-analyze critical-chain` — shows the critical path of the boot sequence (which units were actually blocking the critical path, vs just slow in isolation).

### Real optimization I did

Found `NetworkManager-wait-online.service` taking 5.485s — it blocks boot until the network is confirmed online. I don't have anything that requires network-at-boot, so:
1. Confirmed nothing depends on it: `systemctl list-dependencies network-online.target`
2. Masked it: `sudo systemctl mask NetworkManager-wait-online.service`
3. Rebooted and re-checked with `systemd-analyze` — boot time dropped from 10.538s → 8.415s (2.1s faster).

## Building a custom systemd service — boot-time-check

Wrote a script (`boot-time-check`, see `shell-scripting.md`) that checks userspace boot time against a threshold and fires a desktop notification (`notify-send`) if it's exceeded. Wired it up to run automatically after every boot:

**1. Unit file** — `/etc/systemd/system/boot-time-check.service` (actually placed as a **user** unit, run via `--user` commands below):
```ini
[Unit]
Description=Check userspace boot time and notify if slow
After=graphical-session.target

[Service]
Type=oneshot
ExecStart=/home/tanish/.local/bin/boot-time-check

[Install]
WantedBy=default.target
```

- `After=graphical-session.target` — waits until the GNOME session is fully up
- `Type=oneshot` — runs once and exits, doesn't stay resident
- `ExecStart` — absolute path to the script (script itself lives in `~/.local/bin`)
- `WantedBy=default.target` — makes it run automatically at login

**2. Activate it:**
```bash
systemctl --user daemon-reload            # reload unit definitions
systemctl --user enable boot-time-check.service   # creates the WantedBy= symlink
```

**Known limitation to revisit:** the threshold in the script is currently hardcoded. Plan is to replace it with a rolling average of past boot times instead of a fixed number.
