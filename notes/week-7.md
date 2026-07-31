# Week-7
*27/07/26 - 02/08/26*

## curl command
It is used to transfer data between a system and a server using different network protocols.

- `curl [URL]` like say `curl https://geeksforgeeks.com/` will print the contents of that webpage on stdout. It's not very useful as it is as the content is not formatted.

- - The `-I` flag only fetches headers from the webpage. It is much easier to understand.

- I can write multiple URL's by `curl https://site.{one, two, three}.com`, and numeric sequences using `curl ftp://ftp.example.com/file[1-20].jpeg`.

- `-#` option will display a progress bar, and `--silent` disables it.

- To download files from authenticated FTP server, `curl -u {username}:{password} [FTP_URL]`. If i add a `-T {filename}` flag after password, i can upload files.

- One thing i found pretty cool was that i can send emails using this command. `curl --url [SMPT URL] --mail-from [sender mail] --mail-rcpt [receiver mail] -n --ssl-readq -u {email}:{password} -T [mail_text_file]`. The `[SMTP URL]` is the address of the mail server that will relay the message, like for example for gmail it will be `smtp://smtp.gmail.com:587`.

- curl can also be used to test REST APIs, i can perform GET,POST,PUT operations using this command. example: `curl -X GET https://api.sampleapis.com/coffee/hot` and `curl -X POST -d "key1=value1&key2=value2" https://api.sampleapis.com/coffee/hot`. The `id` flag is used to send data with the request.

## ping command
It is used to test the reachability of a host and measure the time it takes for packets to travel to and from that host.
Running the command `ping www.google.com` yields the following result,
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ ping www.google.com
PING www.google.com (2001:4860:4826:7700::) 56 data bytes
64 bytes from 2001:4860:4826:7700::: icmp_seq=1 ttl=117 time=6.97 ms
64 bytes from 2001:4860:4826:7700::: icmp_seq=2 ttl=117 time=8.30 ms
64 bytes from 2001:4860:4826:7700::: icmp_seq=3 ttl=117 time=8.03 ms
^C
--- www.google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 6.972/7.769/8.303/0.574 ms
```

Can also ping an ip address, `ping 8.8.8.8`

Options:
- `-c COUNT`: specifying the number of ICMP packets to send before stopping.
- `-s SIZE`: sets the size of the sent packets (in bytes).
- `-i INTERVAL`: sets the time between sending packets (in seconds).
- `-f`: sends packets as fast as possible. (for stress testing)
- `-p PATTERN`: Fills packets with specific hexadecimal pattern.

The ping command only shows if an IP packet can reach the host and return, and how long does this take. It does not show any other information like bandwidth, path, etc.

## traceroute command
It is used to track the path that data packets take from the computer to a destination. It shows every "hop" (router or server) it passes through along the way, as long as how long each step takes.
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ traceroute google.com
traceroute to google.com (2404:6800:4013:802::64), 30 hops max, 80 byte packets
 1  2405:201:2:9212:f2ed:b8ff:fe6d:d469 (2405:201:2:9212:f2ed:b8ff:fe6d:d469)  7.861 ms  7.812 ms  7.782 ms
 2  * * *
 3  2405:203:400:100:172:31:0:221 (2405:203:400:100:172:31:0:221)  9.344 ms  23.246 ms 2405:203:400:100:172:31:0:222 (2405:203:400:100:172:31:0:222)  9.287 ms
 4  2405:200:801:200::8a4 (2405:200:801:200::8a4)  9.259 ms 2405:200:801:200::8a8 (2405:200:801:200::8a8)  9.228 ms 2405:200:801:c00::11aa (2405:200:801:c00::11aa)  9.182 ms
 5  2405:200:802:3168:61::7 (2405:200:802:3168:61::7)  9.154 ms  10.011 ms  9.968 ms
 6  * * *
 7  2405:200:801:c00::119c (2405:200:801:c00::119c)  9.245 ms  9.600 ms  9.174 ms
 8  * * *
 9  2001:4860:1:1::331c (2001:4860:1:1::331c)  9.438 ms * 2001:4860:1:1::270 (2001:4860:1:1::270)  12.514 ms
10  * 2001:4860:1:1::331c (2001:4860:1:1::331c)  8.617 ms 2404:6800:8281:300::1 (2404:6800:8281:300::1)  11.037 ms
11  2404:6800:8281:240::1 (2404:6800:8281:240::1)  7.962 ms 2404:6800:8281:200::1 (2404:6800:8281:200::1)  10.705 ms 2404:6800:8285:2c0::1 (2404:6800:8285:2c0::1)  7.602 ms
12  2001:4860:0:1::3fe4 (2001:4860:0:1::3fe4)  8.519 ms 2001:4860:0:1::17d0 (2001:4860:0:1::17d0)  6.118 ms 2001:4860:0:1::8766 (2001:4860:0:1::8766)  6.322 ms
13  2001:4860:0:1::1bae (2001:4860:0:1::1bae)  7.885 ms 2001:4860::c:4004:53cb (2001:4860::c:4004:53cb)  36.779 ms 2001:4860::c:4004:53ca (2001:4860::c:4004:53ca)  10.379 ms
14  2001:4860::9:4001:d9e7 (2001:4860::9:4001:d9e7)  25.601 ms  28.204 ms *
15  2001:4860::9:4001:d9e7 (2001:4860::9:4001:d9e7)  32.511 ms 2001:4860::9:4001:ddce (2001:4860::9:4001:ddce)  28.969 ms 2001:4860:0:1::77c7 (2001:4860:0:1::77c7)  36.329 ms
16  2001:4860:0:1::77bf (2001:4860:0:1::77bf)  31.957 ms 2001:4860:0:1::4c14 (2001:4860:0:1::4c14)  30.090 ms 2001:4860:0:1::77c7 (2001:4860:0:1::77c7)  31.227 ms
17  2001:4860:0:1::b68 (2001:4860:0:1::b68)  46.790 ms 2001:4860:0:1::4c14 (2001:4860:0:1::4c14)  37.198 ms 2001:4860:0:1::4c26 (2001:4860:0:1::4c26)  61.485 ms
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  nv-in-f100.1e100.net (2404:6800:4013:802::64)  35.106 ms  37.720 ms  37.635 ms
```

## SSH (Secure Shell)
SSH is a method for secure remote login from one computer to another. It provides strong authentication, protects communications security and integrity with strong encryption.

The protocol works in client-server model. The SSH client initiates the connection process and uses public key cryptography to verify the identity of the SSH server.

SSH is often used to controlling servers remotely, for managing infrastructure and for transferring files.

SSH allows for port forwarding or tunneling. It allows for sending messages by passing them through multiple other ip addresses/ports. Say a server only accepts requests from other computers within a network, so if i want to send a request from a remote location, i have to pass it through a computer within that network that accepts remote connections which will then pass it to the server.

## Python server
I made a simple server using python. It just listens on a port continuously.
It prints all the access it gets onto the terminal (It's a feature of the python server library).
I put it in the linux-journey folder where all these notes and scripts are.
When i ran `localhost:8000` on a browser, i could access all the files in the directory.

I can even see the port,
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ ss -tlnp 'sport = 8000'
State          Recv-Q         Send-Q       Local Address:Port          Peer Address:Port        Process                                         
LISTEN         0              5                  0.0.0.0:8000               0.0.0.0:*           users:(("python3",pid=13065,fd=3))    
```

I can also use the curl command to access it,
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ curl -X GET localhost:8000/README.md
# linux-journey
This repository is to document my journey in learning linux.
```
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ curl -X GET localhost:8000/notes/
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<style type="text/css">
:root {
color-scheme: light dark;
}
</style>
<title>Directory listing for /notes/</title>
</head>
<body>
<h1>Directory listing for /notes/</h1>
<hr>
<ul>
<li><a href="week-1.md">week-1.md</a></li>
<li><a href="week-2.md">week-2.md</a></li>
<li><a href="week-3.md">week-3.md</a></li>
<li><a href="week-4.md">week-4.md</a></li>
<li><a href="week-5.md">week-5.md</a></li>
<li><a href="week-6.md">week-6.md</a></li>
<li><a href="week-7.md">week-7.md</a></li>
</ul>
<hr>
</body>
</html>
```

    
