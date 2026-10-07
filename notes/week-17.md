# Week-17
*05/10/26-11/10/26*

## Ansible automate for CA certificates

By this point the CA worked. Users logged in with short-lived certificates and the hosts proved themselves with host certificates. But all of it was hand-typed: `docker cp` a public key into the CA, `docker exec` to sign it, `docker cp` the result back, edit config, restart. The host certificates I'd issued on 2026-10-03 expire after 30 days, so this was a recurring job with a deadline. A manual procedure with a deadline is the textbook case for automation.


### The control node and how it reaches the CA

My first instinct was to run Ansible against the CA container with `docker exec`. That fails for two reasons:

- Ansible modules need Python on the target, and the `smallstep/step-ca` image (Alpine) has none.
- Driving containers from inside a container means mounting `/var/run/docker.sock` into it. That is root-equivalent on the host and bypasses every network boundary I'd built.

The better insight: I only used `docker exec` because my PC had no `step` CLI. The CA publishes an **HTTPS API**, and the `step` CLI can sign certificates over it from anywhere. So I built a dedicated `control` container with Ansible and the `step` CLI, and let it call `https://ca:9000`. The CA is used the way it was designed to be used, and nothing needs the Docker socket.

### Placing the control node on the networks

It needs to reach two things: the CA's API and the SSH hosts. That gave three options:

- **`public` + `private` + `ca`:** control would bridge the CA and the servers, which destroys the property that nothing connects them.
- **`public` + `ca`, reaching the servers through the bastion:** the `private` network keeps exactly one way in. This is what I chose.
- **Docker socket:** rejected for the reason above.

The result is that Ansible is just another client of the system I built. It logs in through the bastion with `ProxyJump`, with a CA-signed certificate, like any human would.

### Three trust anchors, not one

This took me a while to untangle:

| Trust anchor | What it protects | Who uses it |
|---|---|---|
| X.509 root certificate | The CA's **HTTPS API** | The `step` CLI, via `step ca bootstrap` |
| User CA public key | **Who may log in** | Servers, via `TrustedUserCAKeys` |
| Host CA public key | **Which servers are genuine** | Clients, via `@cert-authority` |

`step ca bootstrap` downloads the root cert and checks it against a **fingerprint** I supply out of band, then saves the config in the running user's `~/.step`. It has nothing to do with SSH trust.

### Passwords: there are four, and they do different jobs

| Secret | What it protects | Where it appears |
|---|---|---|
| **Vault password** | The file `vault.yml` | `--ask-vault-pass` when running the playbook; never stored in the file |
| **CA provisioner password** | The CA's willingness to sign anything | Stored *inside* `vault.yml`; passed to `step` by the task |
| CA key password | The CA's own private keys | `/home/step/secrets/password`; read by the CA at startup |
| SSH key passphrase | A private key file on disk | `--no-password --insecure` turns it off for the control node's key |

Mixing up the provisioner password and the CA key password caused my first `failed to decrypt JWE: invalid password`. Putting the wrong value in the vault caused the second. The distinction is "who is being authenticated to what".

### Ansible connection plugins

`hosts: control` means "look up `control` in the inventory", and the default transport is SSH. The control container has no sshd, so Ansible would try to SSH to itself and fail. A play meant to configure the machine Ansible runs on should target `localhost` with `connection: local`. That also determines which user the tasks run as, and therefore whose home directory `step ca bootstrap` writes to. Using `become: true` there would have trusted the CA as root while everything else ran as another user.

### Idempotency

A good playbook is safe to run twice. I used two patterns:

- **`creates:`** on commands that leave a marker file, so they run once (the bootstrap).
- **A computed condition** for things that expire. Each host's current certificate is read, and a renewal happens only when it's missing or under 7 days from expiry.

### Vault and `no_log`

Vault keeps the provisioner password out of the repo, but it only protects data at rest. At runtime the password exists in memory, so every task that touches it carries `no_log: true`, otherwise verbose output would print it.

## The implementation

### Compose: the control service

```yaml
control:
  build:
    context: ./control
    dockerfile: dockerfile
  working_dir: /work
  volumes:
    - ./control:/work
    - ./ca:/ca:ro        # read-only: control can read the trust line but not rewrite it
  container_name: control
  networks:
    - public
    - ca
```

### SSH-host image changes

Ansible needs Python on the targets and root to write into `/etc/ssh/host_keys`:

```dockerfile
RUN apt-get update && apt-get install -y openssh-server sudo python3 && rm -f /etc/ssh/ssh_host_*
RUN echo "admin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/admin && chmod 440 /etc/sudoers.d/admin
```

### Inventory, with per-host principals

```yaml
all:
  children:
    ssh_hosts:
      vars:
        ansible_user: admin
        ansible_ssh_common_args: "-F {{ playbook_dir }}/ssh_config_control"
      hosts:
        bastion:
          host_principals: [bastion, localhost]
        server1:
          host_principals: [server1]
        server2:
          host_principals: [server2]
```

A certificate is only valid for the names listed as principals, so each host needs its own list. The bastion also gets `localhost` because that's the name my own client uses to reach it.

### The control node's SSH config

Options passed on the command line don't reach the hidden `ProxyJump` first hop, which is the same lesson as my first bastion setup, so everything goes in a config file:

```
Host bastion server1 server2
    User admin
    IdentityFile ~/.ssh/id_ed25519
    CertificateFile ~/.ssh/id_ed25519-cert.pub
    IdentitiesOnly yes
    UserKnownHostsFile /ca/known_hosts
    StrictHostKeyChecking yes

Host server1 server2
    ProxyJump bastion
```

### Variables

```yaml
# group_vars/all/vars.yml
ca_fingerprint: "<root fingerprint>"
ca_provisioner_password: "{{ vault_ca_provisioner_password }}"
host_cert_path: /etc/ssh/host_keys/ssh_host_ed25519_key-cert.pub
```

```yaml
# group_vars/all/vault.yml  (encrypted with: ansible-vault create)
vault_ca_provisioner_password: "<provisioner password>"
```

The `vault_` prefix convention means anyone grepping the repo can see where the real value lives, while tasks use the plain name.

### Play 1: prepare the control node

```yaml
- name: Prepare the control node
  hosts: localhost
  connection: local
  tasks:
    - name: Trust the CA's HTTPS API
      ansible.builtin.command:
        cmd: step ca bootstrap --ca-url https://ca:9000 --fingerprint {{ ca_fingerprint }}
        creates: "{{ ansible_env.HOME }}/.step/certs/root_ca.crt"

    - name: Check the CA is reachable
      ansible.builtin.command: step ca health
      changed_when: false

    - name: Make sure ~/.ssh exists
      ansible.builtin.file:
        path: "{{ ansible_env.HOME }}/.ssh"
        state: directory
        mode: "0700"

    - name: Create control's key pair (once)
      community.crypto.openssh_keypair:
        path: "{{ ansible_env.HOME }}/.ssh/id_ed25519"
        type: ed25519

    - name: Issue a fresh 8h user certificate
      ansible.builtin.command:
        cmd: >
          step ssh certificate --sign --force
          --provisioner admin --provisioner-password-file /dev/stdin
          --principal admin --not-after 8h
          ansible-control {{ ansible_env.HOME }}/.ssh/id_ed25519.pub
        stdin: "{{ ca_provisioner_password }}"
        stdin_add_newline: false
      no_log: true
```

Two details worth recording. `--provisioner-password-file /dev/stdin` plus the module's `stdin:` option lets the password reach `step` without ever being written to disk. The first run proved `step` accepts that path. And `ansible-control` is the certificate's key ID, which shows up in server logs so automation logins are distinguishable from mine (`tanish`).

### Play 2: renew host certificates

```yaml
- name: Issue and renew host certificates
  hosts: ssh_hosts
  become: true
  gather_facts: false
  tasks:
    - name: Read the host's current certificate
      ansible.builtin.command: ssh-keygen -L -f {{ host_cert_path }}
      register: current
      failed_when: false
      changed_when: false

    - name: Decide whether a renewal is due
      ansible.builtin.set_fact:
        renew: >-
          {{ (force_renew | default(false) | bool) or current.rc != 0 or
             ((current.stdout | regex_findall('Valid: from \S+ to (\S+)') | first
               | to_datetime('%Y-%m-%dT%H:%M:%S')) - now()).days < 7 }}

    - name: Fetch the host's public key to control
      ansible.builtin.fetch:
        src: /etc/ssh/host_keys/ssh_host_ed25519_key.pub
        dest: "/tmp/hostkeys/{{ inventory_hostname }}.pub"
        flat: true
      when: renew | bool

    - name: Sign it with the CA
      ansible.builtin.command:
        cmd: >
          step ssh certificate --sign --host --force
          --provisioner admin --provisioner-password-file /dev/stdin
          {% for p in host_principals %}--principal {{ p }} {% endfor %}
          --not-after 720h
          {{ inventory_hostname }} /tmp/hostkeys/{{ inventory_hostname }}.pub
        stdin: "{{ ca_provisioner_password }}"
        stdin_add_newline: false
      delegate_to: localhost
      become: false
      no_log: true
      when: renew | bool

    - name: Install the certificate on the host
      ansible.builtin.copy:
        src: "/tmp/hostkeys/{{ inventory_hostname }}-cert.pub"
        dest: "{{ host_cert_path }}"
        owner: root
        mode: "0644"
      when: renew | bool
      notify: Reload sshd

  handlers:
    - name: Reload sshd
      ansible.builtin.service:
        name: ssh
        state: reloaded
```

Notes on the mechanics:

- The signing task uses `delegate_to: localhost`, so it runs on the control node, where `step` and the CA trust live, but with each host's variables.
- A **handler** runs once at the end of the play, only if a task notified it. sshd is PID 1 in these containers, so it has to be reloaded, never restarted, or the container dies.
- `force_renew=true` lets me exercise the renewal path on demand instead of waiting for expiry.

Running it:

```bash
docker exec -it control bash
cd /work
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
ansible-playbook -i inventory.yml site.yml --ask-vault-pass -e force_renew=true
```

## What went wrong along the way

| Symptom | Cause | Fix |
|---|---|---|
| `failed to decrypt JWE: invalid password` | Wrong secret: the CA key password in place of the provisioner password, and later the wrong value stored in the vault | Use the provisioner password from `docker logs ca` |
| `Could not match supplied host pattern: ssh_hosts` | Ran the playbook without `-i inventory.yml` | Always pass the inventory |
| `Missing sudo password` | `admin` had no sudo rights, and `become` needs them | Passwordless sudoers entry for `admin` in the image |
| `ssh-keygen` failed on a `C:/.../Git/etc/...` path | Git Bash rewrites arguments starting with `/` | `MSYS_NO_PATHCONV=1` before `docker exec` |
| My own login failed, and a copied cert did nothing | The human certificate (1h) had expired, and I'd also copied it to the wrong filename (`key-cert.pu`) | Re-issue it, copy it to the right name, start an agent in that shell and `ssh-add` again |

A pattern behind several of these: a successful outcome doesn't prove the mechanism did the work. A login worked while the certificate wasn't involved at all, and a task said `changed` without proving sshd was serving the new certificate.

## Verifying it properly

The playbook reporting `changed` on all three hosts wasn't enough. The real check is what a client receives during the handshake:

```bash
ssh -v -F ssh_config server1 true 2>&1 | grep -iE "Server host certificate|known and matches"
```

| Host | Serial before | Serial after | Valid until |
|---|---|---|---|
| bastion | `13764757154211677940` | `326664126737054182` | 2026-11-06 |
| server1 | `15161208515906204963` | `13490649992394516742` | 2026-11-06 |
| server2 | `13613987186099281307` | `5242324572890086345` | 2026-11-06 |

The serials changed on the wire, so the running sshd had reloaded. The host key fingerprints stayed identical, because the keys live on persistent volumes. Only the certificate rotated, which is why my client needed no changes: it trusts the CA, not any per-host fingerprint. Strict checking still passed on every hop.

## What I'd flag as unfinished

- The control node holds the provisioner password at runtime, so it can mint certificates for any principal. It's the most sensitive node in the topology.
- `admin` now has full sudo. That's a deliberate tradeoff, because Ansible modules generally don't work under scoped sudo rules. The mitigation is that `admin` is reachable only with a short-lived certificate.
- Nothing runs the playbook on a schedule, so a renewal still depends on someone running it before 2026-11-06.
- I haven't exercised the "host has no certificate yet" path from a blank state, only forced renewals.
- The provisioner password was pasted into a chat during development, so it's a lab-only value.
- The expired-certificate rejection test is still pending.












