# Project: Bastion Access with an SSH Certificate Authority

Project-specific log. Concepts are in `security/access-control.md` (SSH, ProxyJump, hardening) and `security/ssh-ca.md` (certificates, trust anchors, signing). Automation concepts: `ansible/fundamentals.md`, `ansible/vault.md`.

## Goal

A bastion (jump host) as the single entry point to private servers, with login and server identity both backed by a CA instead of static keys. Then automate certificate issuance and renewal with Ansible.

## Architecture

```
PC --ssh--> bastion (public + private nets) --ssh--> server1, server2 (private only)
control (public + ca nets) --HTTPS--> ca (ca net only; API on 127.0.0.1:9000)
```

- Three Docker networks: `public`, `private`, `ca`.
- The bastion sits in both `public` and `private`. Servers are only in `private`. The CA is isolated in `ca`.
- Rejected: putting `control` on `public` + `private` + `ca` (it would bridge the CA and the servers), and mounting the Docker socket.
- The control node reaches the servers through the bastion with `ProxyJump` and a CA-signed certificate, like any human client.

## Build log

### Week 16: bastion and keys
- Generated an ed25519 key pair; initial setup put the public key in `authorized_keys` on each container.
- `ProxyJump` failed at first: `-i` applies only to the final destination, the hidden first-hop `ssh` found no key and fell back to a password. Fixed with an `ssh_config` per host block.
- Host keys: created at package install, so identical across containers from one image. Fix: delete them in the same `RUN` as the install, regenerate at start via an entrypoint running `ssh-keygen -A` then `exec sshd -D`. Host keys live on named volumes.
- Hardening via a drop-in `hardening.conf` (`PasswordAuthentication no`, `PermitRootLogin no`) pulled in by `Include`; verified with `sshd -T`.

### Week 16: CA, user certs
- `ca` service from `smallstep/step-ca` with `DOCKER_STEPCA_INIT_SSH: "true"`, state in the `ca_data` volume.
- Signed my public key, exported the CA's user public key as `user_ca.pub`, baked it into the image at `/etc/ssh/user_ca.pub`, added `TrustedUserCAKeys`.
- `CertificateFile` added to both `ssh_config` blocks.
- Removed `authorized_keys` provisioning, recreated containers: login still works, the CA signature is the only way in.

### Week 16: host certs
- Built a project-local `known_hosts` with an `@cert-authority` line covering `[localhost]:2222`, `bastion`, `server1`, `server2`.
- `Host *` block: `UserKnownHostsFile ca/known_hosts`, `StrictHostKeyChecking yes`.
- Copied the three public host keys into the CA, signed with 720h, copied certs back to each volume, added `HostCertificate`.
- Bastion cert needed `localhost` as an extra principal.

### Week 17: automation
- Dedicated `control` container (Ansible + `step` CLI) calls the CA's HTTPS API.
- Play 1 prepares the control node (bootstrap CA trust, key pair, 8h user cert). Play 2 renews host certs when missing or under 7 days from expiry, with a handler reloading sshd.
- `admin` got passwordless sudo and the images got Python, because Ansible needs both.
- Verified by watching certificate **serials change on the wire** (before/after per host), host key fingerprints unchanged, strict checking still passing on every hop. New validity: **until 2026-11-06**.

## Unfinished

- Renewal is not scheduled; someone must run the playbook before **2026-11-06**.
- Control node can mint certs for any principal; `admin` has full sudo.
- Not yet exercised: host with no cert from a blank state; expired-cert rejection test.
- The provisioner password was pasted into a chat during development, so treat it as a lab-only value and rotate it before any real use.
