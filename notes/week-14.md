# Week-14
*14/09/26 - 20/06/26*

## Ansible roles
Roles are basically a set of tasks, instead of writing all the tasks for all the nodes in one playbook, roles allow to create separate folder containing tasks and other customizations like variables, templates, etc for a set of tasks. I can then use this role multiple times i.e. on multiple nodes without having to rewrite all the tasks again.

In my ansible project i used it create a role called web that sets up a web server and displays a static page (for now, plan is to later show a full web app). I created 2 docker containers for web and configured the same role on both of them so both display the same web page and i didn't have to write the tasks 2 times.

One more role i created is for a load balancer. It basically is the reason why i created 2 web servers, to implement load balancing. It has tasks to be a server and display the web page from any one of the 2 web servers. I implemented nginx task in it. 

### Templates
It is a folder in the roles. As it name suggests it is a customizable template. Any file put in it ends with `.j2` like `nginx.conf.j2` or `index.html.j2`. 
It allows to customize files based on which node it is being applied to as we assign the role to multiple nodes. 
For example, i used in web role to add a variable `{{ inventory_hostname }}` which resolves to the hostname/node name. I put the index.html in the templates folder, renamed it to `index.html.js` and added the variable in the title. In the tasks when using a template the module to be used is `template`. This caused it so that the web page now showed which node it is coming from, i did this to display the functionality of load balancing. In the templates module in tasks i put `src: index.html.j2` but i also put dest: /var/www/html` which is for a normal html file, this caused an error since i am not just using an html file so nginx cannot recognize it and hence doesn't display it. So i have to remove the `.j2` extension when pasting it in the node by instead putting `dest: /var/www/html/index.html`

I also used a template in the load balancer role. Here i put the nginx config file in the template folder as `nginx.conf.j2`. The `j2` functionality i used here is a for loop. In the upstream block of http block i put 
```
{% for web in groups['web'] %}
    server {{ web }};
{% endfor %}
```
this makes it so it automatically resolves to the nodes under the web group in `inventory.ini` file. This is so that if i add more nodes for web in the future i wont have to change this and it will automatically add the new web server nodes. The `groups['web']` is an in-built variable that lists all the nodes under the specifies group (here web).


### Handlers
This is another folder in a role. Here we can put tasks that we want to run only when a change is made on a machine. 
In the load balancer role i a handler task that reloads nginx. I put this handler task in the task that deploys nginx in the main task in tasks folder by adding `notify: reload nginx`, `reload nginx` is the name of the handler task.


## CI Pipeline
I set up a basic CI pipeline on Github in my ansible devops infra project. It just builds the docker containers and check if the load balancing feature is working or not. I ran into a surprisingly large number of issues and problems for such a small pipeline.

The github workflow executes on a PR (pull request) to the main from from a feature branch. It builds the containers using the docker compose file. I put healthcheck for the control node (container) to ensure that everything is running and the relevant things are installed properly as those are needed to run the future commands in the workflow. It then runs the ansible playbook in the control container which sets up the web containers and the loadbalancer container. Next it verifies that the load balancer is working. 

The load balancer verification took me some time as i could not figure out how to do that. i eventually made a small script (in the workflow file itself) that curls the loadbalancer 10 times and stores the output in a text file. If the text file and the end contains both "web1" and "web2" text then the loadbalancer is working, as i print the container the webpage is coming from in the title so i can check from the webpage where it came from.

A surprising challenge was in designing a good healthcheck for the control container.   In the test field i was using `CMD` which is not suitable for running multiple commands, so i switched to `CMD-SHELL` which allowed me to run multiple commands to check the status of ansible, curl, and sshpass in the container.


Load Balancer, CI Pipeline, and Hardening — Project Journal
Load Balancer + Multi-Host Content

What I did: Built out web1, web2, and loadbalancer as separate containers on a shared Docker network, all provisioned by Ansible from a control container. The web role installs nginx and deploys a templated index.html that renders inventory_hostname, so each node's page identifies itself. The loadBalancer role builds an nginx upstream block dynamically from groups['web'], using least_conn, and proxies to it. Verified it actually works by curling the load balancer repeatedly and watching it alternate between web1 and web2 — real proof, not just "the config looks right."

### Mistakes and fixes:

Used the service module with start as the state param — wrong; it's state, not start.
template/copy tasks with a directory-only dest sometimes kept the .j2 extension on the deployed file instead of dropping it — had to be explicit with the full destination filename.
Tried the systemd module for managing nginx — fails outright, because these containers have no init system; sshd -D (or equivalent) runs as PID 1 with nothing managing child services. Had to use service instead.
Ran apt install without -y in a non-interactive context — it hung indefinitely waiting for confirmation and held the apt lock, breaking every subsequent task until I killed it.

What I learned: Idempotent config management isn't just a buzzword — I felt directly why the service vs systemd distinction matters once "no init system" became a real constraint instead of a hypothetical. Also learned to actually prove functionality (repeated curling) rather than trusting that a clean Ansible run implies correct behavior.

### GitHub Actions CI Pipeline

**What I did:** Built a workflow triggered on PRs that builds the whole stack (docker compose up --build --wait), runs the playbook non-interactively via docker exec control ansible-playbook ..., verifies load balancing with a loop of curl requests asserting both web1 and web2 show up in the responses, and tears down with if: always() so cleanup happens regardless of pass/fail. Set up branch protection requiring the check before merge.

Mistakes and fixes:

- First healthcheck attempt used CMD with a single string containing spaces (["CMD", "ansible --version"]) — CMD doesn't go through a shell, so it tried to execute a literal binary named "ansible --version" and failed immediately. Switched to CMD-SHELL, which does run through /bin/sh -c.
- First attempt at "check multiple binaries are installed" used which sshpass curl ansible — didn't realize which's multi-argument exit-code behavior is inconsistent/implementation-dependent. Switched to command -v ansible >/dev/null && command -v curl >/dev/null && command -v sshpass >/dev/null — POSIX-portable and explicit about requiring all three.
- Initially thought a single curl to the load balancer was "verification" — realized that only proves nginx responded, not that load balancing (alternation between web1/web2) is actually happening. Rebuilt it as a 10-request loop asserting both hostnames appear across the combined output.
- Didn't initially think through what happens to a for loop if one docker exec call fails mid-run — learned that GitHub Actions' default shell invocation includes -e (errexit), so a single failing command kills the whole step immediately rather than letting the loop finish. This is actually the correct behavior for this check (one failed request should fail the whole verification), not something to "fix."

**What I learned:** A CI check needs to assert something specific, not just "did the command exit 0." Also got real experience with why docker exec's pseudo-TTY behavior and non-interactive execution matter in an automated context, versus my own terminal where I never think about it.

## Hardening

What I did: Built a hardening role applied uniformly to all three managed hosts. Generated an SSH keypair, deployed the public key via authorized_key, and — critically — only disabled password authentication after manually proving key-based login worked end-to-end. Also set PermitRootLogin prohibit-password. Used reload (not restart) for sshd changes.

### Mistakes and fixes:

- Private key ended up with 0777 permissions after being bind-mounted into the container — SSH refused to use it and silently fell back to a password prompt, which looked like a totally different bug at first. Fixed by adding a file task (mode: '600', delegate_to: localhost) that runs before any key-based connection attempt, so it self-heals regardless of how the file's permissions land.
- Tried service sshd restart to apply an sshd_config change — this killed the entire container, because sshd runs as PID 1 with no init system, and Docker stops a container the instant its PID 1 exits. Had to use reload instead, which re-reads config in place without the process exiting.
- Got the actual service name wrong at first (sshd vs ssh — Ubuntu's init script is named ssh, not sshd, even though the binary is sshd) — learned not to assume distro conventions and to check directly (ls /etc/init.d/) instead.
- Pointed a lineinfile task at /etc/ssh/ssh_config instead of /etc/ssh/sshd_config — client config instead of server config. The task silently "succeeded" (wrote to a file, just the wrong one) without any error, which was the scariest kind of bug — no failure signal at all, just quietly not doing what I thought it was doing.
- Used PermitRootLogin without password — deprecated syntax; modern OpenSSH wants prohibit-password. Broke the reload entirely (unsupported option), which — combined with the file already being written — briefly left the live config in an invalid state, correctable only because sshd hadn't actually reloaded yet.
- Biggest process lesson: after docker compose down && up to test something unrelated, the fresh containers had no key deployed yet, and I'd already commented out the password fallback in inventory — total lockout (Permission denied (publickey,password)). Had to temporarily re-enable password auth in the inventory to re-bootstrap. This is now a documented, deliberate part of the setup rather than a surprise: fresh containers always need one password-authenticated run before they're key-managed.

### Attempted and deliberately dropped:

- Scoped sudo (replacing NOPASSWD:ALL with a narrow command allowlist) — technically straightforward, but decided it wasn't worth the implementation time relative to its showcase value; documented as a known next step instead of built.
- ufw / fail2ban — attempted, and hit a real architectural wall: both need the NET_ADMIN/NET_RAW kernel capabilities to modify iptables, which Docker containers don't have by default. - Decided not to grant these capabilities just to make the demo work, since doing so would hand each container a much bigger privilege than the security benefit justifies — and in a real deployment, firewalling belongs at the host/cloud layer, not inside individual containers anyway. This ended up being one of the more interesting lessons of the whole project: knowing when not to force a feature to work is itself a real engineering judgment call, not a failure.

**What I learned, overall:** The recurring theme across all of hardening was ordering discipline around anything that can lock you out — deploy-then-verify-then-disable, never disable-then-hope. Also learned that "no error" isn't the same as "did the right thing" (the ssh_config vs sshd_config bug), and that hitting a hard constraint (container capabilities) and correctly deciding not to work around it is a legitimate, documentable outcome — not an unfinished task.

## Ansible Vault
**What I set out to do:**

Add Ansible Vault to the project ahead of the upcoming db/cache tier, so secrets are handled correctly from the start instead of retrofitted later. Scope: encrypt a placeholder secret in the hardening role, prove decrypt works locally, then wire the same mechanism into the GitHub Actions CI pipeline.

**What I actually did:**

Vault setup (local)

Created control/vault_pass.txt holding a password I generated, and added it to .gitignore before creating the file so it could never accidentally get staged.
Created control/roles/hardening/vars/vault.yml containing a placeholder secret:
  `vault_db_password: "some-placeholder-value"`
Encrypted the whole file:
  `ansible-vault encrypt roles/hardening/vars/vault.yml --vault-password-file vault_pass.txt`
Referenced it from the plaintext vars file, so the encrypted value is used via indirection rather than directly:
  # vars/main.yml
  `db_password: "{{ vault_db_password }}"`
Verified decryption independently of running the playbook:
  `PAGER=cat ansible-vault view roles/hardening/vars/vault.yml --vault-password-file vault_pass.txt`
Ran the full playbook locally with the vault flag — succeeded across all three hosts:
  `ansible-playbook -i inventory.ini playbook.yaml --vault-password-file vault_pass.txt`

### CI wiring

Added a GitHub Actions repository secret ANSIBLE_VAULT_PASSWORD.
Added a step to Test_Connection.yml to materialize it on the runner before the playbook step:
yaml
  - name: Write vault password file
    run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > control/vault_pass.txt
Added --vault-password-file vault_pass.txt to the existing playbook step.
Mistakes made and how they were found/fixed

1. Private SSH key had been committed to Git history

While reasoning through where the vault password file should live, checked whether the same "gitignore protects it" assumption actually held for the SSH key, which uses the identical pattern.
Ran git ls-files | grep ssh_keys → both the private key and .pub were tracked, despite a .gitignore entry.
Root cause: the original gitignore pattern was ./control/ssh_keys (leading ./, no trailing slash) — didn't match. Fixed to:
  control/ssh_keys/
Untracked the files going forward (does not erase them from history):
bash
  git rm --cached control/ssh_keys/ansible_hardening_key
  git rm --cached control/ssh_keys/ansible_hardening_key.pub
Correct remediation for a leaked credential is rotation, not history-scrubbing — history rewrites (filter-repo/BFG + force-push) are unreliable once anyone has cloned, and don't guarantee the secret wasn't already seen. Generated a new SSH keypair and treated the old one as permanently compromised.
Committed the .gitignore fix and the untracking together.

2. Private key had Windows CRLF line endings, breaking libcrypto

After rotating the key, ansible-playbook failed with:
  Load key "ssh_keys/ansible_hardening_key": error in libcrypto

— happening before SSH auth is even attempted, so not a permissions/authorization problem.

Diagnosed with:
bash
  ssh-keygen -y -f ssh_keys/ansible_hardening_key   # errored, confirming file-level corruption
  cat -A ssh_keys/ansible_hardening_key | head -5   # showed ^M$ at every line end — CRLF, not LF
Quick fix on disk:
bash
  sed -i 's/\r$//' ssh_keys/ansible_hardening_key
Root cause: git config --get core.autocrlf → true on my Windows host, silently converting LF→CRLF on checkout for a file Git guessed was text.
Durable fix — force Git to treat key files as binary regardless of autocrlf, via .gitattributes:
  ssh_keys/* -text

3. First CI run failed at Gathering Facts — SSH permission denied

docker compose up --build in CI always builds fresh containers with no SSH key deployed and password auth still enabled — unlike my long-running local containers, which already had the key trusted from earlier runs.
Fix: re-uncommented ansible_password in inventory.ini so the very first connection in a fresh CI run can bootstrap via password, exactly as documented as the intended fallback in the hardening role's README.

4. Second CI run failed — SSH key files "absent"

file (/work/ssh_keys/ansible_hardening_key) is absent, cannot continue
Root cause: actions/checkout@v4 clones the repo fresh — and since the key files are (correctly) no longer tracked in Git, a clean checkout simply doesn't have them.
Fix: added two more GitHub Actions secrets (ANSIBLE_SSH_PRIVATE_KEY, ANSIBLE_SSH_PUBLIC_KEY) and a step to reconstruct both files on the runner before docker compose up --build:
yaml
  - name: Restore SSH keys
    run: |
      mkdir -p control/ssh_keys
      printf '%s\n' "${{ secrets.ANSIBLE_SSH_PRIVATE_KEY }}" > control/ssh_keys/ansible_hardening_key
      printf '%s\n' "${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}" > control/ssh_keys/ansible_hardening_key.pub
      chmod 600 control/ssh_keys/ansible_hardening_key
PR passed after this.

**Key lessons for revision**
- .gitignore only affects files not yet tracked — it has zero effect on files already committed; always verify with git ls-files | grep <path> rather than trusting the ignore rule.
- A leaked credential (key, password, token — any of them) should be treated as permanently compromised and rotated, not "cleaned up" via history rewriting.
- --vault-password-file takes a file path, never the secret value directly — GitHub secrets have to be written to a file on the runner first, then referenced by path.
- GitHub Actions runners are ephemeral VMs destroyed after each job — anything written to disk during a run (vault password file, restored SSH keys) disappears automatically; no manual cleanup step is needed.
- core.autocrlf=true on Windows can silently corrupt any committed file whose exact bytes matter (SSH keys, certs) — .gitattributes with -text is the durable fix, independent of any one developer's Git config.
- CI containers are always freshly built — never assume CI has the "already bootstrapped" state your long-running local containers have.

## db tier (MySQL role)

### What I set out to do
Add the db/cache tier to the project, per the roadmap, now that Vault was in place to handle secrets properly from the start. Scoped down to MySQL only, no Redis, no persistent volume — this is an infra showcase, not a real application, so I kept scope tight rather than dragging the project out.

### What I actually did

**Infrastructure additions**
- New `db` service in `docker-compose.yml`, built from the same base Dockerfile as everything else (no image changes needed — the DB gets configured entirely by Ansible, same as nginx on web/loadBalancer).
- New `[db]` group in `inventory.ini`.
- New play in `playbook.yaml`:
  ```yaml
  - hosts: db
    become: yes
    vars_files:
      - roles/db/vars/vault.yml
    roles:
      - db
      - hardening
  ```

**db role tasks**
```yaml
- name: Install Mysql
  apt:
    name:
      - mysql-server
      - python3-pymysql
    state: present
  become: true

- name: Start Mysql
  command: service mysql restart
  become: true
  changed_when: true

- name: Wait for Mysql socket to be ready
  wait_for:
    path: /var/run/mysqld/mysqld.sock
    state: present
    timeout: 30
  become: true

- name: Create application database
  community.mysql.mysql_db:
    name: myapp
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock
  become: true

- name: Create application db user
  community.mysql.mysql_user:
    name: myapp_user
    password: "{{ db_password }}"
    priv: "myapp.*:ALL"
    host: "%"
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock
  become: true
```

**Secrets**
- Moved the DB secret into its own home: `roles/db/vars/vault.yml` (encrypted, holds `vault_db_password`), referenced from `roles/db/vars/main.yml` as `db_password`.
- Installed the required collection on the control node:
  ```bash
  ansible-galaxy collection install community.mysql
  ```

## Mistakes made and how they were found/fixed

**1. `mysql` is not a valid apt package name**
- Corrected to `mysql-server` + `python3-pymysql` (the latter required on the target for the `community.mysql` modules to function at all).

**2. Module auth defaults to TCP, but fresh MySQL only trusts root via Unix socket**
- `mysql_db`/`mysql_user` failed with connection errors until `login_unix_socket: /var/run/mysqld/mysqld.sock` was added to both tasks.

**3. Race condition: tasks ran before mysqld finished initializing**
- Symptom: `Can't connect to MySQL server on 'localhost' (Errno 111 Connection refused)`, later `(Errno 2 No such file or directory)` once other issues were fixed — both were really "the socket file doesn't exist yet."
- mysqld takes several seconds after starting to finish TLS/cert setup and create its socket.
- Fixed with a `wait_for` task polling for `/var/run/mysqld/mysqld.sock` before proceeding.

**4. `service` module's `state: restarted` silently left MySQL down**
- The `service` module reported `changed: true`, but `service mysql status` on the actual container showed "MySQL is stopped."
- Diagnosed by comparing manual testing (`docker exec -it db bash` → `service mysql start` worked fine) against an ad-hoc Ansible command run as the real `ansible` user via sudo — which also worked — versus the module itself, which didn't.
- Root cause: the `service` module implements `restarted` as a separate stop, then a separate start — not the init script's own atomic restart. Under this container's minimal init environment, start fired before stop had fully released MySQL's socket/lock files, so the start silently failed while `changed: true` was still reported (because the stop half did succeed).
- Fixed by bypassing the module's restart abstraction entirely:
  ```yaml
  - name: Start Mysql
    command: service mysql restart
    become: true
    changed_when: true
  ```

**5. Vault variable "undefined" despite the file decrypting correctly**
- First occurrence: `vault_db_password` was defined in the `hardening` role's vars — but `playbook.yaml` ran the `db` role *before* `hardening` for that host, so the variable didn't exist yet when `db`'s tasks needed it.
- Moved the secret into `roles/db/vars/vault.yml`/`vars/main.yml` instead — the structurally correct home, since a role's secrets should live with that role.
- Same error persisted even after the move — because **Ansible only auto-loads a role's `vars/main.yml`, not other files sitting in the same `vars/` folder.** `vault.yml` was never being loaded at all; the reference in the original `hardening` role had been broken the exact same way from day one and nobody had noticed because nothing had ever actually used it.
- Fixed by explicitly loading the file at the play level:
  ```yaml
  vars_files:
    - roles/db/vars/vault.yml
  ```

**6. Inventory ambiguity: group and host both named `db`**
```
[WARNING]: Found both group and host with same name: db
```
- Ansible warns because group vs. host references become ambiguous. Left as-is for now (cosmetic warning, not a functional blocker) — worth renaming the host (e.g. `db1`) later if this bites in a template/hostvars context.

### Key lessons for revision
- `service` module `state: restarted` ≠ an init script's own `restart` — it's two separate operations under the hood, and can race on services with slow shutdown/startup. When in doubt, shell out to the actual init script and mark `changed_when: true`.
- A module reporting `changed: true` is not proof the end state is what you expect — always independently verify (`service <name> status`, or an ad-hoc `shell` command) rather than trusting the module's summary alone when something seems off.
- Ansible only auto-loads `vars/main.yml` per role. Any other file in a role's `vars/` folder — like a `vault.yml` — is inert unless explicitly loaded via `vars_files` (playbook/play level) or `include_vars` (task level). This is an easy, silent mistake, since the file still decrypts fine with `ansible-vault view` — the failure only shows up when something actually references the variable.
- Play-level role ordering matters for variable availability, not just task ordering within a role — a variable defined by role B isn't visible to role A if A runs first in the same play.
- `community.mysql.mysql_db`/`mysql_user` need `login_unix_socket` against a freshly-installed MySQL, since root has no TCP/password auth configured out of the box.
- Diagnose Docker/Ansible service-state mismatches by comparing three vantage points when they disagree: manual `docker exec` shell, an ad-hoc Ansible command as the real connecting user, and the actual module — narrowing down which layer is lying is faster than guessing at the module's abstraction.
