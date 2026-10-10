# Project: Self-Hosted Git Server (Gitea on an Old Android Phone)

Project-specific log. General concepts live in the topic files: `networking/tools.md` (SSH, Termux), `networking/reverse-proxy.md` (Caddy), `networking/dns.md` (DNS, DDNS, certificates), `docker/compose.md`, `security/firewall.md`.

## Goal

Run my own git server without paying for a VPS, using an old Android phone as the host. Stack: Gitea + MySQL in Docker, fronted by Caddy for HTTPS, reachable via a DuckDNS name.

## Timeline

| Week | What happened |
|---|---|
| 8 | Turned the phone into an SSH-reachable server with Termux (static IP, wake lock) |
| 11 | Chose Gitea; wrote a compose file (Gitea + MySQL); learned Caddy as reverse proxy; worked through local-name/certificate trust problems |
| 13 | Compose file tested on the laptop; hit the wall that Docker can't be installed in Termux |

## Phone setup (week 8)

- Installed Termux, `pkg install openssh`, set a password with `passwd`, started `sshd` (port **8022**).
- Installed `iproute2` to get `ip addr show wlan0`; connected with `ssh -p 8022 <termux-user>@<phone-ip>`.
- Switched the phone's Wi-Fi from DHCP to **Static** so the IP survives restarts.
- `termux-wake-lock` so Termux keeps running with the screen off.
- Routine afterwards: open Termux, run `sshd`, `ssh` in from the laptop.
- Known weakness: password authentication on a non-standard port. Candidate for hardening (see `security/access-control.md`).

## Gitea + Caddy design (week 11)

- Compose file defines two services: `gitea` and `mysql` (database storage).
- Request flow: browser -> Caddy (443, HTTPS) -> `gitea:3000` over the internal Docker network. Gitea's own port is never exposed publicly.
- Caddy's local CA was not trusted by my browser because `certutil` wasn't available in the container, so I clicked through a warning during local testing.
- `tanishserver.local` didn't resolve because mDNS needs something actively broadcasting it.
- Local-only options (hosts file, `.local`, LAN IP + self-signed cert) all hit permission or trust problems, and I have no admin access on the company network.
- **Decided direction:** a real domain (DuckDNS) pointing at a publicly reachable server, so Caddy can obtain a real Let's Encrypt certificate.

## Blocker (week 13): no Docker in Termux

- Docker needs kernel features (cgroups, namespaces) that Android kernels are usually compiled without.
- A GitHub repo describes the workaround: run **QEMU** to emulate a full x86_64 machine, install Alpine in it, and run Docker inside that VM.
- That is a lot of moving parts, and the phone becomes an emulator host. Performance and complexity are open concerns.

## Open questions / next steps

- Decide: QEMU + Alpine on the phone, versus a small VPS, versus running Gitea directly (non-Docker) in Termux.
- Public reachability needs port forwarding (80/443) or a tunnel; the DuckDNS DDNS client must keep the record updated.
- Harden SSH on the phone before exposing anything.
- Gitea Actions / CI later.
