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

## Still to cover here

- `iptables` / `nftables` directly (ufw is a simplified frontend over these — worth understanding what it's actually generating underneath)
- `fail2ban` — intrusion prevention by watching logs and banning IPs after repeated failed attempts (e.g. SSH brute force)
