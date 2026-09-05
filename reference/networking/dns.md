# DNS & Name Resolution

How hostnames turn into IP addresses, and what actually "proves" domain ownership.

## /etc/hosts

A static lookup table mapping hostnames to IPs, consulted **before** any DNS server is reached.

```
IPAddress        Hostname             Alias
127.0.0.1        localhost            deep.openna.com
208.164.186.1    deep.openna.com      deep
208.164.186.2    mail.openna.com      mail
```

As a machine boots, it needs some hostname-to-IP mappings before DNS is even available — this file provides that. In the absence of a nameserver, any network program falls back to this file to resolve a hostname.

## /etc/resolv.conf

The configuration file for DNS **queries** — stores the IP address(es) of nameservers to query.

- I initially assumed `curl` looked in `/etc/hosts` for the DNS server's address — wrong. It looks in `/etc/resolv.conf`; `/etc/hosts` is just a fallback/default lookup every command checks first regardless.
- The **resolver** is the library of functions that does the actual domain-to-IP translation by querying nameservers, and `/etc/resolv.conf` is what configures it.
- A **nameserver** is a DNS server authoritative for (or capable of answering about) certain domains. The `nameserver` directive in this file specifies which one(s) to query.

**How does this file get populated automatically?**
When a machine joins a network, it does a **DHCP handshake** with the router. The DHCP response includes: IP address, gateway, and one or more DNS server IPs — which is how `/etc/resolv.conf` gets filled in without manual config, wherever I connect.

## DDNS (Dynamic DNS)

Keeps a DNS record updated with a host's current IP address, even when that IP keeps changing (e.g. no static IP from an ISP).

Flow:
1. A DDNS client (built into a router, or standalone software) detects a new IP address on the device.
2. It sends an update request to the primary DNS server.
3. The request contains the hostname (e.g. `myhome.dyndns.org`) and the new IP.

Used **DuckDNS** (a free DDNS provider) as the domain for my self-hosted server since I don't have a static public IP.

## How domain ownership is actually verified (Let's Encrypt / ACME)

This was a genuine "aha" while setting up Caddy + TLS for my Gitea server:

- **Caddy itself does not verify domain ownership — DNS does.**
- If my domain's **A record** points at my server's IP, then when Let's Encrypt performs its HTTP-01 challenge, the verification request lands on *my* server — because DNS says that's where the domain lives. That's the actual proof of control.
- Caddy's role is just automating the request for the cert and serving the challenge response. Trust is rooted in **DNS control**, not in anything Caddy does locally.

## localhost vs .local vs LAN IP — and why none of these work for sharing with others

Ran into this directly while trying to make my Gitea server reachable by more than just myself:

- **`localhost`** always means "this machine talking to itself," regardless of who types it — it can never reach another computer. Fine for testing on my own box, useless for anyone else.
- **`.local` hostnames** rely on **mDNS** (multicast DNS), which has to be actively broadcast by something like Avahi/Bonjour — just writing `myserver.local` in a Caddyfile doesn't make it resolvable. This is exactly why `tanishserver.local` gave "site can't be reached": a container has no way to broadcast mDNS to the host/network on its own.
- **A raw LAN IP** works for other devices on the same network, but Caddy's self-signed cert is issued for `localhost`, not the IP — so every device shows a "connection not private" warning, since nothing outside my own machine trusts Caddy's local CA by default.
  - Related: Caddy's logs showed it couldn't auto-install its root cert into the browser trust store because `certutil` wasn't available in the container — hence having to manually click through the warning. The connection was still encrypted; nothing had just *vouched* for the cert to the browser.

**The actual fix for sharing beyond my own machine:** all local workarounds (hosts file, `.local`, self-signed LAN IP) hit permission/trust walls — and I don't have admin access on the company network anyway. The real answer is a **real domain** (even a free one like DuckDNS) pointed at a **publicly reachable server**, so Caddy can get a genuine, automatically-trusted Let's Encrypt certificate with zero manual trust steps for anyone connecting.
