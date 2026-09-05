# Networking Fundamentals

Core concepts: sockets, netfilter, and basic diagnostic tools.

## Sockets

A socket is a software endpoint enabling bidirectional communication between processes, regardless of whether they're on the same machine or different ones.

**Two types:**
1. **Network sockets** — communication across networks, using protocols like TCP/IP.
2. **Domain sockets** (Unix domain sockets) — communication between processes within the same system.

## ss (Socket Statistics)

Displays detailed info about network sockets — modern replacement for `netstat`.

```
tanish@...:~$ ss
Netid   State   Recv-Q   Send-Q   Local Address:Port   Peer Address:Port
u_str   ESTAB   0        0        * 97273              * 97274
```

**Column meanings:**
- **State** — socket status: `LISTEN` (waiting for a connection) or `ESTABLISHED` (active communication).
- **Recv-Q / Send-Q** — amount of data queued for receiving/sending.
- **Local Address:Port** — IP/port on my system where the socket is listening or created.
- **Peer Address:Port** — the remote system's IP/port connected to my machine.

**Useful flags:**
- `ss -s` — summary of all socket types (totals by protocol/family):
  ```
  Total: 1110
  TCP:   22 (estab 12, closed 6, orphaned 0, timewait 5)
  Transport  Total  IP   IPv6
  TCP        16     10   6
  ```
- `ss -tlnp 'sport = 8000'` — show TCP listening sockets on a specific port, with the owning process. Used this to confirm a Python server was actually bound to port 8000:
  ```
  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
  LISTEN  0       5       0.0.0.0:8000        0.0.0.0:*          users:(("python3",pid=13065,fd=3))
  ```

## netfilter

A kernel subsystem providing the framework for packet filtering, NAT, and connection tracking. It works via **hooks** — points in the kernel's network code where functions can register to run for specific network events. `ufw` and `iptables` are both built on top of this.

### The 5 netfilter hooks

Think of these as checkpoints at different stages of a packet's journey through the network stack:

1. **`NF_IP_PRE_ROUTING`** — triggered by incoming traffic immediately after it enters the network stack, **before** any routing decision is made.
2. **`NF_IP_LOCAL_IN`** — triggered after an incoming packet has been routed and is destined for the local system.
3. **`NF_IP_FORWARD`** — same as above, but for packets being forwarded to another host.
4. **`NF_IP_LOCAL_OUT`** — triggered by locally-generated outbound traffic as soon as it hits the network stack.
5. **`NF_IP_POST_ROUTING`** — triggered by outgoing or forwarded traffic after routing, just before it goes out on the wire.

## ping

Tests reachability of a host and measures round-trip time.

```bash
ping www.google.com
ping 8.8.8.8
```
```
64 bytes from ...: icmp_seq=1 ttl=117 time=6.97 ms
--- www.google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 6.972/7.769/8.303/0.574 ms
```

**Options:**
- `-c COUNT` — number of ICMP packets to send before stopping
- `-s SIZE` — packet size in bytes
- `-i INTERVAL` — time between packets (seconds)
- `-f` — flood ping, sends as fast as possible (stress testing)
- `-p PATTERN` — fills packets with a specific hex pattern

**Limitation to remember:** `ping` only tells you if a packet can reach the host and return, and how long that took — nothing about bandwidth, path, or anything else.

## traceroute

Shows the path packets take to a destination, hop by hop (each router/server along the way), plus the time for each hop.

```bash
traceroute google.com
```
```
traceroute to google.com (...), 30 hops max, 80 byte packets
 1  <hop1-ip>  7.861 ms  7.812 ms  7.782 ms
 2  * * *
 3  <hop3-ip>  9.344 ms  23.246 ms ...
 ...
23  nv-in-f100.1e100.net (...)  35.106 ms  37.720 ms  37.635 ms
```
- `* * *` means that hop didn't respond (often a router configured to not reply to traceroute probes — not necessarily a failure).
