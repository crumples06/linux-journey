# Week-15
*21/09/26 - 27/09/26*

# Project Journal — Observability Milestone (node_exporter, Prometheus, Grafana)

## Goal for this session

Move on to the last major infra milestone: metrics collection and visualization. Landed on the standard node_exporter → Prometheus → Grafana stack, with node_exporter applied as a cross-cutting concern (like `hardening`) and Prometheus + Grafana bundled into a new `monitoring` role on a dedicated host.

## What got built

- **`node_exporter` role**: downloads the v1.10.2 tarball, installs the binary, and — since these containers have no systemd — a hand-written SysV-style init script using `start-stop-daemon` to background the process and track it via a pidfile. Applied to every host.
- **`monitoring` role, Prometheus half**: tarball install, config/data directories, a templated `prometheus.yml` that builds its scrape-target list straight from Ansible inventory groups (`groups['web'] + groups['loadBalancer'] + groups['db']`) — same pattern the `loadBalancer` role already used for its nginx upstream block. Same init-script approach as node_exporter, reused directly.
- **`monitoring` role, Grafana half**: apt-repo install using the newer `deb822_repository` module (handles GPG key fetching/formatting itself — no manual `gpg --dearmor` step needed). Admin password stored via Ansible Vault, same pattern as the db role's `vault_db_password`. Datasource and the community "Node Exporter Full" dashboard (ID 1860) both auto-provisioned via config files Grafana scans on startup — no manual UI clicking.
- Decided to **skip Alertmanager** — a real fourth service with its own install complexity, for marginal benefit over a purely visual demo. Documented as a deferred future improvement instead.

Verified end to end: all four node_exporter targets showing `up` in Prometheus, Grafana admin login working, Prometheus datasource live, dashboard rendering.

## Bugs, in the order I hit them

**Prometheus tarball missing `consoles`/`console_libraries`.** Most install guides describe these being copied alongside the binary — they were removed from the Prometheus release tarball as of v3.x, and the docs I was following just hadn't caught up. Deleted the two copy tasks, no replacement needed. First real lesson of the session: check what's actually in the tarball rather than trusting a guide's file list.

**Prometheus init script had CRLF line endings.** Same root cause as the SSH-key corruption incident from a few sessions back — a Windows-origin file with `\r\n` line endings instead of `\n`. Showed up as a misleading `service prometheus start` → "No such file or directory," which is actually an execve failure on a shebang line ending in `\r`, not a missing-file problem at all. `cat -A` showed the `^M` characters; `sed -i 's/\r$//'` fixed it. Worth broadening the `.gitattributes` entry from just `ssh_keys/*` to cover every role's `files/` directory so this class of bug can't recur project-wide.

**YAML indentation bug in the templated `prometheus.yml.j2`.** Each nested level was indented 4 spaces deeper than its parent instead of sibling keys (`job_name`, `static_configs`) aligning at the same column — plus a `job name`/`job_name` typo on top. Produced a `did not find expected key` parse error. Diagnosed by running the `prometheus` binary directly instead of through the init script (bypasses the backgrounding, so the error prints straight to the terminal), then `cat -n` on the rendered file to actually see the indentation rather than guessing from the template source.

**Grafana shipped no SysV init script.** Assumed it would, based on older install docs — this apt-repo build turned out to be systemd-unit only. Caught by checking `dpkg -L grafana` and `find` directly rather than continuing to trust the docs, which was the right instinct after the Prometheus tarball surprise. Read the actual `grafana-server.service` unit file and hand-translated its `ExecStart`, `EnvironmentFile`, `User`/`Group`, and `RuntimeDirectory` into a new `start-stop-daemon` init script.

**Grafana pidfile permission denied.** `start-stop-daemon --make-pidfile` wrote the pidfile as root, before the process dropped privileges to the `grafana` user via `--chuid`. Grafana then tried to write its *own* pidfile (it manages this internally, unlike node_exporter/Prometheus) as `grafana`, and couldn't overwrite a root-owned file. Fixed by dropping `--make-pidfile` and letting Grafana handle its own pidfile entirely — the mismatch was two different things trying to own the same file.

**Datasource provisioning permission denied.** The provisioning YAML was deployed with `mode: "0640"` but no explicit `owner`/`group`, so it landed `root:root`. The `grafana` user isn't in the `root` group, so it couldn't read its own datasource config. This turned out to be the same underlying issue as the pidfile bug — root creates something, a non-root service user needs it later, and only the mode bits were considered, not ownership.

**Admin login stuck on the wrong password.** Renamed a variable (`admin_password` → `grafana_admin_password`) to match convention but only updated one of the two places it was referenced, so the template rendered an undefined var as empty and Grafana silently fell back to its own built-in default (`admin`/`admin`). Fixing the naming mismatch didn't fix login, though — turns out `admin_password` in `grafana.ini` is only read once, at first-ever admin account creation. Once the SQLite user record already exists, changing the config and restarting does nothing. Had to wipe the container and let it re-bootstrap fresh. Genuinely non-obvious behavior, worth remembering for any future Grafana work outside this project too.

**Datasource silently not registering** — `/api/datasources` returned `[]` with nothing but a vague deprecation warning in the logs. Root cause was the templated file rendering completely empty (0 bytes) rather than any content or permission problem this time. A reminder that "no error, but nothing happened" is its own category of bug worth checking file contents for directly, rather than assuming the error log will always point at it.

**`grafana cli plugins ls` failing** on a missing `/var/lib/grafana/plugins` directory. Same root-cause family as the earlier `/run/grafana` gap — a directory normally created invisibly by the apt package's postinst script or by systemd, absent here because there's no init system doing that setup work. Added an explicit `file` task, same fix pattern as before.

## The throughline

Almost every bug this session traced back to one of two things: **docs/guides describing an older version of the software than what I actually had** (Prometheus's consoles, Grafana's init script), or **things systemd/apt normally do invisibly that had to be replicated by hand** in a no-init-system container (runtime directories, pidfile ownership, the plugins directory). Neither is really an Ansible problem — Ansible did exactly what it was told each time. The debugging always came down to reading the actual state of the system (`dpkg -L`, the systemd unit file, `cat -n` on a rendered config, log files) rather than trusting what a guide said should be there.

## What's left

- Kill-and-recover demo (stop nginx on web1, screenshot dashboard/load-balancer behavior, restart, screenshot recovery) — not yet run.
- Incident/runbook writeup for that demo.
- Top-level architecture README tying all five roles together.

Feeling fairly worn down by this point in the project — made the call to skip Alertmanager and keep the demo visual-only specifically to get to the finish line rather than chase one more piece of infrastructure.
