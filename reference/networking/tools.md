# Networking Tools

curl, SSH, and hands-on server setup notes.

## curl

Transfers data between a system and a server over various protocols.

**Basics:**
- `curl [URL]` — prints page contents to stdout (unformatted, not very readable on its own).
- `-I` — fetch headers only.
- Multiple URLs / ranges:
  ```bash
  curl https://site.{one,two,three}.com
  curl ftp://ftp.example.com/file[1-20].jpeg
  ```
- `-#` — show progress bar; `--silent` — hide it.
- Authenticated FTP: `curl -u {username}:{password} [FTP_URL]`, add `-T {filename}` after the password to **upload** a file.
- Can send email via SMTP:
  ```bash
  curl --url [SMTP_URL] --mail-from [sender] --mail-rcpt [receiver] -n --ssl-reqd -u {email}:{password} -T [mail_text_file]
  ```
  (Gmail's SMTP URL: `smtp://smtp.gmail.com:587`)
- REST API testing:
  ```bash
  curl -X GET https://api.sampleapis.com/coffee/hot
  curl -X POST -d "key1=value1&key2=value2" https://api.sampleapis.com/coffee/hot
  ```
  `-d` sends data with the request.

### What actually happens when I run `curl <url>`

1. Parses the URL, determines the protocol (http, ftp, etc).
2. Asks the system resolver to turn the hostname into an IP — checks `/etc/hosts` first, then `/etc/resolv.conf`'s nameservers, possibly hitting a local DNS cache along the way. (See `dns.md`.)
3. Opens a TCP connection to that IP on the target port.
4. Sends a plain HTTP request over that connection — headers (`-H`), cookies (`-b`), body (`-d`) all get folded in here.
5. Server responds: status line (`HTTP/1.1 200 OK`), headers, body.
6. curl reads the response — prints body to stdout by default. Relevant flags:
   - `-i` — include response headers
   - `-o file` — write to a file instead of stdout
   - `-L` — follow redirects
   - `-v` — verbose, shows the full exchange including the TLS handshake
7. Connection closes, or is kept alive/reused if `Connection: keep-alive`.

## SSH (Secure Shell)

Method for secure remote login between computers, with strong authentication and encrypted, integrity-protected communication.

- Client-server model: the SSH client initiates the connection and uses **public key cryptography** to verify the server's identity.
- Common uses: remote server control, infrastructure management, file transfer.
- **Port forwarding / tunneling** — lets a connection be relayed through intermediate hosts/ports. Example use case: if a server only accepts connections from within its own network, a remote client can reach it by first connecting to a machine *inside* that network which is allowed to talk to the server, then tunneling through it.

## Running a simple Python server

Made a basic server with Python's built-in server library — just needs to be run in the directory to serve.

- Logs every access to the terminal automatically (built into the library).
- Visiting `localhost:8000` in a browser lists/serves files in that directory.
- Can confirm it's listening with `ss` (see `fundamentals.md`):
  ```bash
  ss -tlnp 'sport = 8000'
  ```
- Can hit it with curl:
  ```bash
  curl -X GET localhost:8000/README.md
  curl -X GET localhost:8000/notes/   # returns an auto-generated directory listing page
  ```

## Turning an old Android phone into a server (Termux)

Wanted a self-hosted server without paying for a VPS — repurposed an old Android phone instead.

**Setup steps:**
1. Install **Termux** (terminal emulator + Linux environment for Android).
2. Inside Termux: `pkg install openssh` — needed to connect in via SSH.
3. Set a password: `passwd`.
4. Start the SSH daemon: `sshd`. Termux's `sshd` runs on **port 8022** (not the usual 22) — important to remember when connecting. It backgrounds itself automatically.
5. Get the phone's IP: `ip addr show wlan0` — required installing `iproute2` first (`pkg install iproute2`) since `ip` wasn't available by default. Look for the `inet <ip-address>` line.
6. From the laptop, connect with:
   ```bash
   ssh -p 8022 u0_a651@<phone-ip>
   ```
   (`u0_a651` is the Termux-generated username.)
7. Set the phone's Wi-Fi to **Static** instead of DHCP, so the IP doesn't change every time the server restarts. After this, the routine is just: open Termux → `sshd` → `ssh` in from the laptop.
8. Run `termux-wake-lock` so Termux keeps running even when the screen is off (otherwise Android would likely suspend it).

Can also just `ping` the phone directly to confirm reachability on the LAN before trying SSH.
