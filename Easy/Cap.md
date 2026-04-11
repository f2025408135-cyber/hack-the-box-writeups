---
# Hack The Box: Cap

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP** | 10.10.10.245 |
| **Points** | 20 |
| **Release Date** | June 2021 |
| **Retired Date** | October 2021 |
| **My Rating** | 5/10 |

## TL;DR
Discovered a network dashboard running on port 80 that allowed downloading PCAP files. An Insecure Direct Object Reference (IDOR) vulnerability let me download a PCAP from before my session, which contained plaintext FTP/SSH credentials for the user `nathan`. After SSHing in, I checked Linux capabilities and found the Python 3.8 binary had `cap_setuid` enabled, allowing me to trivially spawn a root shell.

## Reconnaissance
I started this box with my standard, thorough port scan to see what services were exposed.

```bash
nmap -sC -sV -p- -T4 --min-rate 1000 10.10.10.245 -oA nmap/initial
```
<!-- screenshot: nmap initial scan results -->

The scan returned three open ports relatively quickly:
*   **Port 21:** FTP (vsftpd 3.0.3)
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (gunicorn)

Since Anonymous FTP login was disabled (I checked manually with `ftp 10.10.10.245`), I directed my attention to the web server on port 80. 

I opened my browser and navigated to `http://10.10.10.245`. The site was a security dashboard providing various network and system metrics. The fact that it was running on `gunicorn` indicated a Python backend, likely Flask or Django.

<!-- screenshot: web application landing page -->

I clicked around the dashboard. Most of the links just showed static or generic system information. However, the "Security Snapshot" (or similarly named) feature caught my eye. 

This feature allowed the user to download a network packet capture (PCAP) file, presumably for analysis. 

## Initial Foothold
When I clicked the button to generate and download a PCAP, the application redirected me to a URL that looked like this:

`http://10.10.10.245/data/3`

The number `3` at the end of the URL is a massive red flag. Predictable, sequential numeric identifiers in URLs are the textbook definition of a potential Insecure Direct Object Reference (IDOR) vulnerability.

An IDOR occurs when an application provides direct access to objects based on user-supplied input without properly verifying authorization. If I could access `/data/3`, what would happen if I changed that number?

I manually modified the URL in my browser to access previous captures.

`http://10.10.10.245/data/0`

The server didn't throw a 403 Forbidden or a 404 Not Found error. Instead, it happily served me a PCAP file named `0.pcap`. This file was generated *before* my session even began, meaning it contained data captured from some previous interaction or automated script on the server.

This was the breakthrough. I didn't need to fire up `ffuf` or Burp Intruder to brute-force a massive list of IDs; the very first one (`0`) was all I needed.

I downloaded `0.pcap` to my attacking machine and opened it in Wireshark.

```bash
wireshark 0.pcap &
```

The PCAP file wasn't huge, but looking through raw packets can be tedious. I immediately filtered the traffic to look for low-hanging fruit: unencrypted protocols. 

I applied the Wireshark filter `ftp` (or `tcp.port == 21`). 

The filter immediately highlighted a cleartext FTP session. I right-clicked on one of the packets and selected "Follow -> TCP Stream" to see the entire conversation.

```text
220 (vsFTPd 3.0.3)
USER nathan
331 Please specify the password.
PASS Buck3tH4TF0RM3!
230 Login successful.
```

The PCAP analysis yielded valid, plaintext credentials for the user `nathan`.

A common mistake in security (and a frequent theme in CTFs) is password reuse. While these credentials were used for FTP, I knew SSH was also open on port 22.

I immediately tried to authenticate via SSH using the discovered credentials.

```bash
ssh nathan@10.10.10.245
# Password: Buck3tH4TF0RM3!
```

<!-- screenshot: successful exploitation (reverse shell) -->

The connection was successful. I had my initial foothold as the user `nathan`. I grabbed the `user.txt` flag from his home directory.

## Privilege Escalation
With a stable SSH session, I started my standard Linux privilege escalation checks.

First, I checked `sudo -l` (required a password, but `nathan` had no special `sudo` rights). I checked for SUID binaries using `find / -perm -4000 2>/dev/null` and didn't see anything obviously vulnerable or out of place.

My next step is always to check Linux capabilities. Capabilities are a way to grant specific elevated privileges to an executable without giving it full root access (like a SUID bit does). However, if an administrator grants an overly permissive capability to a powerful binary, it can be abused.

```bash
getcap -r / 2>/dev/null
```

<!-- screenshot: sudo -l output -->
*(In this case, the screenshot would be of the getcap output)*

The output scrolled by, and one line stood out brilliantly:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

The `/usr/bin/python3.8` executable had the `cap_setuid` capability assigned to it. This is incredibly dangerous. The `cap_setuid` capability allows a process to manipulate its own UID (User ID). This means the Python binary, even when executed by a standard user like `nathan`, can arbitrarily change its UID to 0 (root).

I didn't need to compile any C code or run a complex exploit script. I could just use Python's built-in `os` module to elevate my privileges.

I executed the Python binary and ran a quick one-liner to set the UID to 0 and spawn a system shell (`/bin/bash`).

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

The prompt changed immediately.

```bash
id
# uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

<!-- screenshot: root flag -->

Got root! That was a fun one. The IDOR to PCAP analysis felt very realistic, and checking capabilities is a crucial skill that is often overlooked in favor of just running LinPEAS.

## Lessons Learned
- **The Danger of IDOR:** Predictable, sequential identifiers (like `/data/3`) are a massive security risk if authorization checks are not rigorously enforced on every single request. An attacker can simply enumerate the IDs to access sensitive data belonging to other users or the system itself.
- **Unencrypted Protocols:** FTP transmits credentials in plaintext. Capturing network traffic containing FTP (or Telnet, HTTP basic auth, etc.) will expose those credentials to anyone analyzing the PCAP. Always use secure alternatives like SFTP or SSH.
- **Overly Permissive Capabilities:** Granting the `cap_setuid` capability to a powerful scripting language binary like Python effectively gives any user on the system the ability to become root. Capabilities must be assigned with extreme caution and only to narrowly scoped, single-purpose binaries.
- **What tripped me up:** At first, I tried to use `ffuf` to brute-force a massive list of IDs for the PCAP download, which was unnecessary and noisy. The very first ID (`0`) contained the credentials I needed. Sometimes, the simplest approach is the best.
- **Pro Tip:** When analyzing a PCAP file in Wireshark, don't just look at the raw packets. Always use the "Follow TCP/UDP Stream" feature on interesting protocols (HTTP, FTP, Telnet) to quickly read the conversation as plaintext.

## References
- Insecure Direct Object Reference (IDOR) Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html
- GTFOBins (Python Capabilities): https://gtfobins.github.io/gtfobins/python/#capabilities
- Official Hack The Box Page: https://app.hackthebox.com/machines/Cap
