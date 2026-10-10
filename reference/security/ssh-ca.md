# SSH Certificate Authority (user & host certificates)

How SSH certificates work and how to run a CA for them (Smallstep `step-ca`). Project context and exact commands run: `projects/bastion-ssh-ca.md`. SSH basics and hardening: `security/access-control.md`.

## Why certificates instead of `authorized_keys` / `known_hosts` entries

- With plain keys, every server must list every user's public key, and every client must remember every server's fingerprint.
- With a CA, servers trust **one** user CA key and clients trust **one** host CA key. Anyone with a certificate signed by the CA gets in; any server with a signed host certificate is recognised.
- Certificates can be **short-lived** (1h, 8h), which limits damage from a stolen key.

## The two directions

| | Signs | Proves | Configured on |
|---|---|---|---|
| **User CA key** | A user's *public* key -> user certificate | "This person may log in" | Servers: `TrustedUserCAKeys /etc/ssh/user_ca.pub` |
| **Host CA key** | A server's *public* host key -> host certificate | "This server is genuine" | Server: `HostCertificate ...`; clients: `@cert-authority` line in `known_hosts` |

Only public keys move around. Signing creates a `-cert.pub` file next to the key.

## Principals

A certificate is valid only for the names listed as **principals**.
- User cert: `--principal admin` means it can log in as the Unix user `admin`.
- Host cert: the client compares the hostname it typed against the cert's principals. If I connect to the bastion as `localhost`, the cert needs `localhost` as a principal as well as `bastion`.

## Three trust anchors, not one

| Trust anchor | What it protects | Who uses it |
|---|---|---|
| X.509 root certificate | The CA's **HTTPS API** | The `step` CLI, via `step ca bootstrap` |
| User CA public key | **Who may log in** | Servers, via `TrustedUserCAKeys` |
| Host CA public key | **Which servers are genuine** | Clients, via `@cert-authority` |

`step ca bootstrap --ca-url https://ca:9000 --fingerprint <fp>` downloads the root cert, verifies it against a fingerprint supplied **out of band**, and saves config in the running user's `~/.step`. It has nothing to do with SSH trust.

## Running step-ca in Docker

- Image: `smallstep/step-ca`. `DOCKER_STEPCA_INIT_SSH: "true"` makes first start create two SSH signing keys (user and host).
- Keep its state in a named volume so the CA keeps its identity across restarts.
- Put it in its own network, away from the servers. Publish its API on `127.0.0.1` only.
- The CA is an **HTTPS API**: the `step` CLI can sign over the network from any container that has it. I only used `docker exec` at first because my PC had no `step` CLI.
- Driving containers with Ansible over `docker exec` is a trap: the Alpine CA image has no Python, and mounting `/var/run/docker.sock` is root-equivalent on the host.

## Signing

User certificate (short-lived):
```bash
step ssh certificate --sign --provisioner admin --principal admin --not-after 1h tanish /tmp/key.pub
```
Host certificate (one per host, with that host's principals):
```bash
step ssh certificate --sign --host --provisioner admin \
  --principal bastion --principal localhost --not-after 720h bastion bastion_host.pub
```
- The last two arguments are the certificate's **key ID** and the public key to sign. The key ID shows up in server logs, so give automation a distinct one (`ansible-control` vs `tanish`).
- The provisioner password is requested. `--provisioner-password-file /dev/stdin` lets automation pipe it in without writing it to disk.
- Output lands next to the input as `<name>-cert.pub`.

## Wiring it up

Servers (`sshd` hardening drop-in):
```
TrustedUserCAKeys /etc/ssh/user_ca.pub
HostCertificate /etc/ssh/host_keys/ssh_host_ed25519_key-cert.pub
```
Clients (`ssh_config`), in **every** host block because `ProxyJump` opens two separate connections:
```
CertificateFile keys/key-cert.pub
```
Client trust of host certs (project-local `known_hosts`):
```
@cert-authority [localhost]:2222,bastion,server1,server2 <host CA public key>
```
- `[host]:port` is how OpenSSH writes a host on a non-standard port. The pattern must match how the client actually connects.
- `Host *` with `UserKnownHostsFile` pointing at that file and `StrictHostKeyChecking yes` makes the client refuse anything it cannot verify, with no prompt to accept.

Once certificates work, delete `authorized_keys` provisioning. The CA signature becomes the only way in.

## Verifying (don't trust "it logged in")

A login can succeed with the certificate not involved at all (fallback key, wrong file). Check what is actually presented:
```bash
ssh -v -F ssh_config server1 true 2>&1 | grep -iE "Server host certificate|known and matches"
ssh-keygen -L -f <cert-file>       # inspect principals, validity, serial
```
- After a renewal, the certificate **serial changes on the wire** while the host key fingerprint stays the same (keys persist on volumes, only the certificate rotates). That proves the running sshd reloaded.
- Clients need no change after renewal: they trust the CA, not a per-host fingerprint.

## Expiry and renewal

- Host certs issued for 30 days are a recurring job with a deadline. Automate it (`ansible/fundamentals.md`: computed `renew` condition, handler to reload).
- sshd as PID 1 in a container must be **reloaded**, never restarted.
- If a human cert expires you simply get refused; re-issue, copy to the **correct filename**, and `ssh-add` again in that shell if using an agent. Check the filename: I once copied a cert to `key-cert.pu`.

## Things that go wrong

| Symptom | Cause |
|---|---|
| `failed to decrypt JWE: invalid password` | Wrong secret (CA key password vs provisioner password) |
| Cert present but ignored | Wrong filename, or `CertificateFile` missing from one of the two `ProxyJump` blocks |
| Host check fails on the bastion | `localhost` not in the bastion cert's principals |
| Git Bash mangles a `/path` argument | Set `MSYS_NO_PATHCONV=1` before `docker exec` |

## Still unfinished / worth hardening

- The control node holds the provisioner password at runtime, so it can mint certificates for any principal: the most sensitive node.
- `admin` has full passwordless sudo because Ansible modules generally don't work under narrow sudo rules. Mitigation: `admin` is reachable only with a short-lived certificate.
- Nothing runs renewal on a schedule yet.
- The "host has no certificate yet" path and the expired-certificate rejection test haven't been exercised.
