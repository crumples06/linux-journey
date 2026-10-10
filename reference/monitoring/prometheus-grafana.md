# Monitoring: node_exporter, Prometheus, Grafana

The standard metrics stack: node_exporter exposes host metrics, Prometheus scrapes and stores them, Grafana visualises them. Project context: `projects/ansible-devops-infra.md`.

## Roles of each piece

| Piece | Job | Default port |
|---|---|---|
| node_exporter | Exposes host metrics (CPU, memory, disk, network) over HTTP | 9100 |
| Prometheus | Scrapes targets on an interval, stores time series, serves queries | 9090 |
| Grafana | Dashboards and datasources on top of Prometheus | 3000 |

node_exporter runs on **every** host (a cross-cutting concern, like hardening). Prometheus and Grafana live together on a dedicated monitoring host.

## Prometheus config built from inventory

The scrape target list is generated from Ansible groups, the same trick the load balancer upstream block uses:
```jinja
{% for host in groups['web'] + groups['loadBalancer'] + groups['db'] %}
```
Add a host to the inventory and it is scraped automatically.

YAML indentation is the usual failure here. Sibling keys (`job_name`, `static_configs`) must align at the same column. Symptom: `did not find expected key`. Debug by running the binary in the foreground so errors print to the terminal, then `cat -n` the **rendered** file instead of reading the template.

## Grafana provisioning (no UI clicking)

Grafana scans provisioning directories on startup:
- Datasource YAML -> Prometheus.
- Dashboard provider config + dashboard JSON -> e.g. community "Node Exporter Full" (ID 1860).

Verify the datasource registered: `GET /api/datasources` returning `[]` means it did not.

## Lessons

1. **Guides go stale; check what is actually installed.**
   - Prometheus v3.x tarballs no longer contain `consoles`/`console_libraries`, though older guides copy them.
   - The Grafana apt package ships a **systemd unit only**, no SysV init script.
   - Check with `tar -tzf`, `dpkg -L <package>`, `find`, and by reading the unit file.
2. **A service started by root but run as another user needs ownership, not just mode bits.**
   - `start-stop-daemon --make-pidfile` writes the pidfile as root *before* `--chuid` drops privileges, so the service cannot overwrite it. If the service manages its own pidfile (Grafana does), drop `--make-pidfile`.
   - A provisioning file deployed with `mode: "0640"` and no `owner`/`group` lands `root:root`; the `grafana` user cannot read it. Always set `owner`, `group` and `mode` together.
3. **Without systemd/apt scripts, directories must be created by hand.** `/run/grafana` and `/var/lib/grafana/plugins` normally appear invisibly via postinst or systemd `RuntimeDirectory`. Add explicit `file` tasks.
4. **Grafana's `admin_password` in `grafana.ini` is read only once**, at first-ever admin account creation. After the SQLite user record exists, editing the config and restarting does nothing. Reset the password with Grafana's own tooling or wipe state and re-bootstrap.
5. **Update every reference when renaming a variable.** Renaming `admin_password` to `grafana_admin_password` in one place left the template rendering empty, and Grafana silently fell back to `admin`/`admin`.
6. **"No error, nothing happened" is its own bug class.** The datasource silently failed to register because the templated file rendered as 0 bytes. Check file contents directly.
7. **CRLF line endings in a script break the shebang.** `service prometheus start` reported "No such file or directory", which is really an `execve` failure on `#!/bin/sh\r`. `cat -A` shows `^M`; fix with `sed -i 's/\r$//'`. See `git.md` for the `.gitattributes` fix.

## Init scripts without systemd

Containers here have no init system, so each daemon gets a SysV-style script using `start-stop-daemon` (background the process, track via pidfile). See `process-management.md`.

## Deferred

- **Alertmanager** skipped: a fourth service with its own install complexity for little benefit in a visual demo.
- Kill-and-recover demo, runbook and architecture README were still outstanding.
