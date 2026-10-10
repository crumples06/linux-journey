# Firewall

## ufw (Uncomplicated Firewall)

Defines rules for controlling network traffic. Built on top of `iptables` (see `iptables-nftables.md` once that's covered) — provides a simpler interface over the same underlying netfilter framework described in `networking/fundamentals.md`.

**Defaults:**
- Denies **all incoming** connections by default.
- Allows **all outgoing** connections by default.

**Critical gotcha before enabling:**
- If connected over SSH, **allow SSH first** — `sudo ufw allow OpenSSH` — before running `sudo ufw enable`. Otherwise the default-deny-incoming rule locks out the very SSH session being used to configure it, with no way back in remotely.

**Common commands:**
```bash
sudo ufw allow OpenSSH          # allow SSH before enabling, always
sudo ufw enable                 # turn ufw on
sudo ufw status                 # current ruleset
sudo ufw status verbose         # more detail

sudo ufw allow from <IP>        # allow traffic from a specific IP
sudo ufw deny from <subnet>     # deny traffic from a subnet
```

## Firewalls inside containers: why `ufw`/`fail2ban` fail

I tried `ufw` and `fail2ban` in the Ansible project's containers and hit an architectural wall:
- Both need to modify iptables, which requires the **`NET_ADMIN`** and **`NET_RAW`** kernel capabilities.
- Docker containers don't have those by default (and the kernel's netfilter belongs to the host, see `networking/fundamentals.md`).

I could have granted the capabilities to make the demo work, but chose not to:
- It hands each container a much bigger privilege than the benefit justifies.
- In a real deployment, firewalling belongs at the **host or cloud layer** (host `ufw`, cloud security groups), not inside each container.

Knowing when **not** to force a feature to work is a legitimate engineering decision and worth documenting, not an unfinished task.

Related: container-level isolation is also done with networks, not rules: `docker/networking.md`.


## Still to cover here

- `iptables` / `nftables` directly (ufw is a simplified frontend over these — worth understanding what it's actually generating underneath)
- `fail2ban` — intrusion prevention by watching logs and banning IPs after repeated failed attempts (e.g. SSH brute force), attempted in containers; needs NET_ADMIN/NET_RAW. Try on a real host or the Termux phone server
