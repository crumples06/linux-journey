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




