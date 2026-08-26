# Week-11
*24/8 - 30/8*

## Self hosted git server
I decided to make a self hosted git server on my phone. I am using Gitea to set it up. 
It is running on docker, for a start the docker-compose file contains the gitea and a mysql database for storage.


## caddy
Caddy is a web-server. i started learning it to make my self hosted git server. It has automatic TLS/HTTPS for all sites it serves. I can run gites but it exposed on a port without any security or reverse proxy, caddy provides that. 

`caddy run` starts the server in terminal and `caddy start` starts it in the background. 

Caddy is configured primarily in two ways: the Caddyfile (simple, human-friendly) and JSON (powerful, machine-friendly). 
Loading a new config: `curl localhost:2019/load -X POST -H "Content-Type: application/json" --data @caddy.json`
Caddy runs an HTTP API (by default on localhost:2019) that lets you change configuration without restarting.

Today I learned how Caddy works as a reverse proxy in front of a Dockerized app, using my self-hosted Gitea instance as the real example.

**Core concept:** Caddy sits in front of your actual application and handles HTTPS, while your app just does its normal job on its own internal port. Requests flow like this: browser → Caddy (port 443, HTTPS) → your app (e.g. port 3000, internal).

**Ports 80/443 vs. my app's port:** 80 and 443 are Caddy's ports — the "front door" the internet uses to reach it. Port 80 is also used for the ACME HTTP-01 challenge, which is how Let's Encrypt verifies domain ownership before issuing a certificate. My app's own port (3000) never needs to be exposed to the internet at all once Caddy is in front of it — Caddy reaches it internally over the Docker network using the service name (e.g. reverse_proxy gitea:3000).

**How Caddy knows a domain is "mine":** This was one of my bigger questions. It turns out Caddy itself doesn't verify ownership — DNS does. If my domain's A record points to my server's IP, then when Let's Encrypt runs its HTTP-01 challenge, its verification request lands on my server (because DNS says so), proving I control it. Caddy just automates requesting the cert and serving that challenge — trust is rooted in DNS control, not in Caddy.

### localhost vs .local vs LAN IP — and why none of them scale to other people:

localhost always means "this machine talking to itself" — no matter who types it, it never reaches another computer. Great for testing on my own machine, useless for sharing.
.local hostnames rely on mDNS, which needs to be actively broadcast (Avahi/Bonjour) — just writing it in a Caddyfile doesn't make it resolvable, which is exactly why tanishserver.local gave "site can't be reached." A container has no way to broadcast mDNS to the host/network on its own.
A raw LAN IP works for others on the same network, but Caddy's self-signed cert (issued for localhost, not the IP) causes a "connection not private" warning on every device, since nothing outside my own machine trusts Caddy's local CA by default.

**Why Caddy's own local CA didn't get trusted automatically:** The logs showed Caddy couldn't install its root cert into the browser trust store because certutil wasn't available in the container. This is why I had to manually click through the "not private" warning — the cert was valid and encryption was working, but nothing had vouched for it to my browser.

**The real fix for sharing beyond my own machine:* All of the local workarounds (hosts file, .local, raw LAN IP with a self-signed cert) run into permission or trust problems — and I don't have admin access on the company network anyway. The actual production-grade answer is a real domain (even a cheap one, or a free option like DuckDNS) pointed at a publicly reachable server (a small VPS), which lets Caddy get a real, automatically-trusted Let's Encrypt certificate with zero manual trust steps for anyone.
