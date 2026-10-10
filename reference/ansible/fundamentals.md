# Ansible Fundamentals

Concepts and lessons from building a multi-node Ansible project (see `projects/ansible-devops-infra.md` for the project itself).

## Playbook vs role

- A **playbook** is orchestration: it maps hosts (or groups) to roles.
- A **role** is a folder of reusable pieces. Its `tasks/main.yml` is the actual task list, spliced into the play at runtime.
- Roles let the same set of tasks run on many nodes without rewriting them (e.g. one `web` role applied to `web1` and `web2`).

Typical role layout:
```
roles/web/
  tasks/main.yml
  templates/index.html.j2
  files/            # static files copied as-is
  handlers/main.yml
  vars/main.yml
```
Application files belong inside the role (`files/` or `templates/`). This breaks down for large real apps; there, use git-clone-on-target, pre-built artifacts, or containerized apps instead of copying files.

## Inventory

Hosts are grouped in `inventory.ini`:
```ini
[web]
web1
web2

[loadBalancer]
loadbalancer

[db]
db
```
- `groups['web']` is a built-in variable listing every host in that group.
- `inventory_hostname` resolves to the current node's name.
- Warning: a group and a host with the same name (`db`) triggers "Found both group and host with same name". Cosmetic, but rename the host (e.g. `db1`) if it ever causes ambiguity.

## Templates (Jinja2)

- Files in `templates/` end in `.j2` and are rendered per host with the `template` module.
- **The destination must be the full file name without `.j2`**: `dest: /var/www/html/index.html`. With a directory-only `dest`, the file can land with the `.j2` extension and nginx won't serve it.
- Using a variable: `<title>{{ inventory_hostname }}</title>` makes each node's page identify itself.
- Looping over a group so config follows the inventory automatically:
  ```jinja
  upstream backend {
      least_conn;
  {% for web in groups['web'] %}
      server {{ web }};
  {% endfor %}
  }
  ```
  Adding a new web node to the inventory updates the load balancer config with no template edits. The same pattern builds Prometheus scrape targets: `groups['web'] + groups['loadBalancer'] + groups['db']`.

## Handlers

Tasks that run **only when notified by a changed task**, and once at the end of the play however many tasks notified them.
```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: reload nginx
```
```yaml
# handlers/main.yml
- name: reload nginx
  service:
    name: nginx
    state: reloaded
```

## Connection and targeting

- `hosts: X` looks X up in the inventory, and the default transport is SSH.
- To configure the machine Ansible runs on, use `hosts: localhost` with `connection: local`. Otherwise Ansible tries to SSH to itself.
- `connection: local` also decides which user the tasks run as, and so whose home directory things like `step ca bootstrap` write into. Using `become: true` on a local play can configure root instead of the user you meant.
- `delegate_to: localhost` runs one task on the control node while still using the current host's variables (used to sign certs per host).

## Idempotency

A playbook should be safe to run twice. Two patterns:
- `creates:` on a `command` task that leaves a marker file, so it runs once.
- A computed condition for things that expire, e.g. renew only when a cert is missing or under 7 days from expiry (`when: renew | bool`).

Also useful: `changed_when: false` on read-only commands, and an extra var like `-e force_renew=true` to exercise a path on demand.

## Modules and gotchas

- `service` takes `state: started|stopped|restarted|reloaded`. `start` is not a parameter (I once wrote `start: ...`).
- Order tasks logically: deploy the page *before* starting nginx.
- `apt` in non-interactive runs needs `-y` semantics. A bare `apt install` waits for confirmation forever and holds the apt lock, breaking every later task.
- `service` vs `systemd` module: in containers with no init system (sshd or similar is PID 1), the `systemd` module fails outright. Use `service`.
- **`service` with `state: restarted` is two operations (stop, then start)**, not the init script's own `restart`. On services with slow shutdown it can race: start fires before stop releases the socket/lock, MySQL stays down, yet the module reports `changed: true`. Fix: `command: service mysql restart` with `changed_when: true`.
- A module saying `changed` is not proof the end state is right. Verify independently (`service <name> status`, an ad-hoc command).
- When a module and reality disagree, compare three vantage points: a manual `docker exec` shell, an ad-hoc Ansible command as the real connecting user, and the module itself.
- `lineinfile` pointed at the wrong file (`ssh_config` instead of `sshd_config`) succeeds silently. "No error" is not "did the right thing".
- `community.mysql.*` modules need `python3-pymysql` on the target and `login_unix_socket` on a fresh MySQL (root has no TCP/password auth by default). Install the collection with `ansible-galaxy collection install community.mysql`.
- Wait for slow daemons with `wait_for` (e.g. the MySQL socket path) instead of racing them.
- Services that normally get directories, pidfiles and users set up invisibly by systemd or apt postinst scripts need those created explicitly with `file` tasks in a no-init container.
- Pass secrets to commands via `stdin:` (with `--provisioner-password-file /dev/stdin` style options) so they never touch disk, and mark those tasks `no_log: true`.

## Useful commands

```bash
ansible-playbook -i inventory.ini playbook.yaml
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
ansible-playbook ... -e force_renew=true
```
Always pass `-i`. Forgetting it gives "Could not match supplied host pattern".
