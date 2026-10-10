# Project: Ansible DevOps Infrastructure

Project-specific log. Concepts and lessons are in `ansible/fundamentals.md`, `ansible/vault.md`, `ci-cd/github-actions.md`, `monitoring/prometheus-grafana.md`, `security/access-control.md`, `security/firewall.md`, `git.md`.

## Goal

An infrastructure showcase, not a real application: provision a small multi-tier environment entirely with Ansible from a control container, prove it works, harden it, and monitor it.

## Topology

All nodes are Docker containers on one shared network, built from the same base Dockerfile and configured only by Ansible.

| Node | Role(s) | Purpose |
|---|---|---|
| `control` | (runs Ansible) | Control node; has Ansible, curl, sshpass |
| `web1`, `web2` | `web`, `hardening`, `node_exporter` | nginx serving a templated page that shows the node name |
| `loadbalancer` | `loadBalancer`, `hardening`, `node_exporter` | nginx `upstream` (least_conn) built from `groups['web']` |
| `db` | `db`, `hardening`, `node_exporter` | MySQL with an app database and user |
| monitoring host | `monitoring`, `node_exporter` | Prometheus + Grafana |

Containers have **no init system**; sshd (or similar) is PID 1. That drove many decisions (`service` not `systemd`, `reload` not `restart`, hand-written init scripts).

## Build log

### Weeks 13-14: web tier, load balancer
- `web` role: install nginx -> deploy `index.html` -> start and enable nginx. Ran against `web1`/`web2` with `changed=3` on both.
- Verified by curling the containers, not by trusting a clean run.
- Two identical web nodes were deliberate: they make load balancing and failover demonstrable.
- SSH/auth chain: `useradd` -> `chpasswd` -> sshd in the target image -> `sshpass` + `ansible_password` in the inventory -> `NOPASSWD:ALL` sudoers entry for non-interactive privilege escalation.

### Week 14: CI
- PR workflow builds the stack, runs the playbook, curls the load balancer 10 times and asserts both `web1` and `web2` appear. Teardown with `if: always()`. Branch protection on `main`.

### Week 14: hardening
- Key-based SSH: deploy public key via `authorized_key`, prove key login works manually, **then** disable password auth. `PermitRootLogin prohibit-password`. `reload` sshd, never restart.
- Attempted and dropped: scoped sudo (documented as a next step) and `ufw`/`fail2ban` (need `NET_ADMIN`/`NET_RAW`; chose not to grant those capabilities).

### Week 14: Vault
- Vault password file (gitignored), encrypted `vault.yml`, CI secret `ANSIBLE_VAULT_PASSWORD`.
- Found the SSH private key had been committed to git history; rotated the key, fixed `.gitignore`, added `.gitattributes`. Details in `git.md`.

### Week 14: db tier
- MySQL only (no Redis, no persistent volume, scope kept tight). Role: install `mysql-server` + `python3-pymysql`, start MySQL, wait for socket, create DB and user.
- Secret moved to `roles/db/vars/vault.yml`, loaded with `vars_files`.

### Week 15: observability
- `node_exporter` v1.10.2 on every host; Prometheus + Grafana in a `monitoring` role. Datasource and dashboard 1860 provisioned from files.
- Verified: all four node_exporter targets `up`, Grafana login works, dashboard renders.
- Alertmanager skipped on purpose.

## Bugs, by theme

| Theme | Examples |
|---|---|
| No init system | `systemd` module fails; `service sshd restart` killed the container (PID 1); service name is `ssh` on Ubuntu, not `sshd` |
| Silent success | `lineinfile` edited `ssh_config` instead of `sshd_config`; `service` restart reported `changed` while MySQL was down |
| Variable loading | `vault.yml` never auto-loaded; role order hid a variable |
| Ownership/permissions | Private key `0777` after bind mount made SSH ignore it; Grafana pidfile and datasource file owned by root |
| Line endings | CRLF in an SSH key and an init script |
| Stale docs | Prometheus consoles missing; Grafana ships no init script |
| Process lessons | `docker compose down && up` after commenting out the password fallback caused a total lockout |

## Known rough edges (intentional)

- Password-based SSH bootstrap with a username-as-password, and passwordless sudo: the "before" state for hardening, still needed for fresh containers.
- Group and host both named `db` in the inventory (warning only).
- Real app deployment strategy (git clone on target, artifacts, containers) deliberately deferred.

## Remaining

- Kill-and-recover demo: stop nginx on `web1`, capture dashboard/load-balancer behaviour, restart, capture recovery.
- Incident/runbook writeup for that demo.
- Top-level architecture README tying the five roles together.
- Possible later: scoped sudo, Alertmanager, Redis/persistent volume, host-layer firewall.
