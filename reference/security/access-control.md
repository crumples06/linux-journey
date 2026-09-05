# Access Control

Securing remote access to a system — SSH-focused for now, since that's the primary way I access my self-hosted server.

## SSH as an access-control surface

Core protocol mechanics (public-key auth, port forwarding, client-server model) are covered in `networking/tools.md` — this file is specifically about **locking it down**, since SSH is the main entry point into my self-hosted setup and therefore the main thing worth hardening.

**What I know so far, from setting up firewall rules around it:**
- Whenever a firewall (`ufw`) is being configured on a box accessed over SSH, SSH access must be explicitly allowed *before* the firewall is turned on — otherwise the default deny-incoming policy cuts off remote access with no way back in. See `firewall.md`.
- My Termux/Android server setup (`networking/tools.md`) currently uses SSH with a plain password (`passwd` inside Termux) on a non-standard port (`8022`). Non-standard port is a mild deterrent but not real security — this is a candidate for hardening.

## Still to cover here

This section is intentionally a placeholder — SSH hardening is next on my learning roadmap. Topics to fill in once covered:
- Disabling password authentication in favor of key-only login
- Disabling root login over SSH
- Changing/restricting the listening port more deliberately
- `fail2ban` for automatically banning IPs after repeated failed login attempts (ties in with `firewall.md`)
- Restricting SSH access by user/group or source IP/subnet
