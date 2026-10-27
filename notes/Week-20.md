# Week-20
*26/10/26-01/11/26*

## Segmented-Bastion-CA miscellaneous concepts
I was revisiting the bastion project code and I realized that I don't remember everything about that code, and also I had not written any comments in it for me to understand quickly. So I decided to go through the entire code again and write comments in it and also put down anything else that I might have forgotten in the journal.

### Dockerfile
First the dockerfile for `bastion`, `server1`, `server2`. 
Installing `openssh-server` auto-generates host keys, so every container from this image will have the same host keys i.e. `bastion`, `server1`, and `server2` will have the same host keys. So I have to delete those keys and recreate them later in each container separately so every container has a separate key.

I also created the /var/run/sshd directory as it often not present in containers and i need it for ssh.
After creating the `admin` user i have to set the permissions of the file `/etc/sudoers.d/admin` to 440 as sudo silently ignores any sudoers file whose permissions arent 440.

I copied the `host_keygen.sh` script in `/usr/local/bin` because `/usr/local/` is specifically for locally installed software, things the distro's package manager doesn't own. 

I copied `hardening.conf` in `etc/ssh/sshd_config.d/hardening.conf`. Ubuntu's `/etc/ssh/sshd_config` starts with `Include /etc/ssh/sshd_config.d/*.conf`, so this file is loaded before the main config, meaning drop-ins take precedence over later directives in `sshd_config`.

I copied the `user_ca.pub` (the CA user certificate) in `/etc/ssh/user_ca.pub`. 
Normally sshd authenticates users by checking `~/.ssh/authorized_keys` which is a list of exact public keys that may log in, that scales badly. 
SSH certificates invert it. Instead of listing which keys are allowed, the server lists who is allowed to sign keys. `user_ca.pub` is the public half of a signing keypair i.e. an SSH user certificate authority (the `ca` container). The private half, held by the operator (`ca` container), signs short-lived certificates for individual users. The server accepts any certificate bearing a valid signature from that CA, subject to the constraints baked into the certificate.
So, any certificate that is signed by the private key corresponding to `user_ca.pub` is accepted by the server.
For this to work i have to set `TrustedUserCAKeys /etc/ssh/user_ca.pub` in `hardening.conf`.

### hardening.conf
`HostKey /etc/ssh/host_keys/ssh_host_{rsa,ecdsa,ed25519}_key` tells `sshd` which private keys to use as this server's identity. The server signs the key-exchange transcript with one of these so the client can verify it's talking to the right host.

`HostCertificate /etc/ssh/host_keys/ssh_host_ed25519_key-cert.pub`:
Normally a client learns a server's host key on first connect and pins it in `known_hosts`, but my entrypoint regenerates host keys per container, so every docker compose up --build produces a new fingerprint.
At container start, `host_keygen.sh` generates `ssh_host_ed25519_key` and signs its public half with the host CA. `sshd` loads the cert via `HostCertificate` and presents key + cert during the handshake. The client, if it has `@cert-authority` in `known_hosts`, verifies the cert's CA signature and the principal match — and accepts without any prior knowledge of this particular key. 

### control dockerfile Issues
While going through the `control` container's dockerfile, i found some issues. 
I had `RUN ssh-keygen -A` which is redundant as i install `openssh-server` which automatically generated keys on install. 
The second more serious issue is that this bakes in the shared host key as each time the container is generated it will have the same key as the key is baked into the image because of the install. I avoided this problem in the bastion and servers by just deleting the keys and generating them again in the container. 
The control container doesnt need sshd. Control is an SSH client only. It reaches the bastion on port 22 and the servers via ProxyJump. Nothing in the compose file, inventory, playbook or CI connects to the control container over SSH. CI uses docker compose exec.XPOSE 22 and the sshd CMD give it a listening service nobody uses, which is attack surface on a project whose whole point is minimal, hardened access. It also isn’t in known_hosts, has no host certificate, and has no hardening drop-in, so it’s the one SSH daemon in the lab that doesn’t follow the rules.

So, i made some changes,
- Replaced `openssh-server` with `openssh-client` (the Ansible ssh connection needs the client, and `ssh-keygen` ships with it).
- Deleted `RUN ssh-keygen -A`, `RUN mkdir -p /run/sshd`, and `EXPOSE 22`.
- The container still needs a foreground process, since it’s currently kept alive by sshd. Used `CMD ["sleep", "infinity"]` so docker compose exec keeps working.


### ssh_config_control
This is the SSH client config the control container uses. Ansible doesn’t have its own SSH settings here. It just runs the normal ssh binary with this file.
`User admin` to log in as admin, the user certificate’s principal is also admin, and sshd checks that they match. 
It also includes the private key and the certificate that was generated in the playbook. 
`UserKnownHostsFile /ca/known_hosts` is the only source of host trust, bind-mounted read-only from ./ca. It holds the single @cert-authority line.
I also use the host private key and signed certificate (signed public key by ca) using `IdentityFile` and `CertificateFile` respectively.

### Segmented-bastion-ca build from scratch
I realized that the project depends a lot on the `ca` setup, so i was wondering how long and how many steps does it actually take to run the project into a new pc for a person who cloned this project form my repository. I already did all these steps for the CI but i wanted to run them myself.

The steps the need to be done are:
#### Step 1: Clear the original author's material
Need to delete `ca/*`, both files in `group_vars/all`, and `keys/key.pub`

#### Step 2: Start the CA and capture its secrets
Start the `ca` container and read the provisioner password from it's logs and compute the root certificate’s fingerprint.

#### Step 3: Export the CA’s public keys
Copy the user CA and host CA public keys into ca/, and build ca/known_hosts with one @cert-authority line.

#### Step 4: Build and start everything else
Build the SSH image and the control image, and starts the bastion, both servers and the control node.

#### Step 5: Write the Ansible variables and the vault
- `vars.yml` holds the fingerprint, the certificate path, and `ca_provisioner_password: "{{ vault_ca_provisioner_password }}"`.
- `.vault_pass` holds the password that unlocks the vault.
- `vault.yml` holds the CA provisioner password, then ansible-vault encrypt encrypts it.

#### Step 6: First playbook run (bootstrap)
The playbook ran with `ssh_config_bootstrap`, which uses `StrictHostKeyChecking accept-new` and a scratch known-hosts file.
Strict checking needs host certificates, but the playbook is what creates them. `accept-new` is used for this one run only, and it’s passed through `-e` because extra `vars` override the inventory’s `ansible_ssh_common_args`.

#### Step 7: Second run with strict checking
The same playbook, with the normal strict config.

#### Step 8: Your own user key and certificate
Generate a key, copy its public half into the CA container, sign it for principal admin with a 1-hour lifetime, copy the certificate back, and load the key into ssh-agent.

This was extremely tedious so i decided to automate this process as much as i can as i did in the CI pipeline.

### Automation
To automate the majority of the startup i used bash scripts stored in `scripts/`.

#### Stage 1: Removing the root causes

CA password from `.env`. I set `DOCKER_STEPCA_INIT_PASSWORD: ${CA_PASSWORD:?...}` in the compose file, so the CA starts with a password I generated, not a random one shown once in its logs. The `:?` guard turns a missing value into a readable failure instead of an empty password. Side effect: every docker compose command, including down, needs `CA_PASSWORD` to parse the file. That bit me at reset time, covered below.

Mount `user_ca.pub` instead of baking it in. I deleted the `COPY ./ca/user_ca.pub` line from the Dockerfile and added a read-only bind mount on each SSH host. This removes the need to rebuild images when the user CA changes, and the image no longer contains lab-specific material. The cost is that the file must exist before the containers start, or Docker creates a directory in its place. The script guarantees the order.

Stop committing lab-specific files. I gitignored `ca/`, `keys/`, `vars.yml`, `vault.yml` and the generated bootstrap config, and untracked them with `git rm --cached`. After that, a fresh clone has nothing stale to clear, and a success can only be explained by the lab’s own CA.

Smaller changes. `.gitattributes` forces LF on shell scripts, because CRLF breaks the container entrypoint with a bad interpreter error on Windows. I deleted `docker-compose.ci.yml`, since `.env` now does its job.

#### Stage 2: scripts/bootstrap.sh

I moved the CI workflow’s setup logic into a script that is safe to rerun. In order:

1. Generate secrets once. If `.env` doesn’t exist, write a random CA password and a random vault password into it, then load it.
2. Start the CA alone and wait for it to be healthy. Everything downstream depends on its output.
3. Export public material. Copy the user CA and host CA public keys into `ca/`, build `ca/known_hosts` with a single `@cert-authority` line, and read the root fingerprint. Reading the fingerprint from the CA I just created is the one trust decision I’m happy to automate in a lab. In a real deployment it would be recorded out of band.
4. Generate the Ansible variables. `vars.yml` gets the fingerprint, the certificate path, and a reference to the vault variable. `vault.yml` gets the provisioner password.
5. Build and start everything else.
6. Encrypt the vault. The vault password file is written inside the control container, not on the bind mount, and `ANSIBLE_VAULT_PASSWORD_FILE` points at it. This sidesteps a Windows problem I hit earlier, where bind-mounted files show up as executable and Ansible tries to run the password file as a script.
7. Run the relaxed bootstrap playbook only if some host has no certificate. The first run has a chicken-and-egg problem: strict host checking needs host certificates, but the playbook creates them. So that one run uses a config with accept-new and a scratch known-hosts file.
8. Run the playbook again with strict checking. This is the proof. It only succeeds if the hosts now present CA-signed certificates the control node accepts.

#### Stage 3: login.sh and reset.sh

`login.sh` replaces the `docker cp` / `docker exec` routine I’d repeat every hour. It generates my key if needed, has the CA sign it as a 1-hour user certificate for principal admin, copies the certificate back, and loads the key into ssh-agent. A `KEY_ID` variable sets the certificate’s key ID, which matters for CI.

`reset.sh` runs `docker compose down -v` and deletes everything generated: `.env`, the CA’s exported files, the vars and vault, my keys. It passes a dummy `CA_PASSWORD` to down, because compose has to parse the file before it can tear anything down.

#### Stage 4: One path for humans and CI

The workflow shrank from about fifteen setup steps to this:

1. Write `.env` with the CI password.
2. Start the CA and lower the minimum SSH certificate duration on the throwaway CI CA, so the expiry test takes seconds. The lab CA is untouched.
3. Run `bootstrap.sh`.
4. Run `login.sh` with `KEY_ID=ci-user`.
5. Run the positive tests, then the negative tests.

CI now exercises exactly the path a human follows, including the vault, so the README and the pipeline can’t drift apart. The workflow ran to completion on a blank runner.
