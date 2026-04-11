---
# Hack The Box: Beep

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP** | 10.10.10.7 |
| **Points** | 20 |
| **Release Date** | March 2017 |
| **Retired Date** | August 2017 |
| **My Rating** | 4/10 |

## TL;DR
Started with a massive nmap scan revealing a fully loaded Elastix VoIP server. I exploited a classic LFI in the `vtigercrm` component to extract the plaintext `AMPMGRPASS` from the FreePBX configuration. This password turned out to be valid for the `root` user via SSH directly. Alternatively, for privesc, I could have used an interactive `nmap` shell via `sudo`.

## Reconnaissance
I started this box with an aggressive, all-ports scan to ensure I didn't miss any obscure services running on non-standard ports.

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.7 -oA nmap/allports
```

The results came back showing a system absolutely loaded with services. It looked like a unified communications or PBX server out of the box.

```bash
nmap -sC -sV -p 22,25,80,110,111,143,443,993,995,3306,4190,4445,4559,5038,10000 10.10.10.7 -oA nmap/targeted
```
<!-- screenshot: nmap initial scan results -->

The targeted scan confirmed my suspicions:
*   **Port 22:** SSH (OpenSSH 4.3)
*   **Port 80/443:** HTTP/HTTPS (Apache httpd 2.2.3 CentOS)
*   **Port 25/110/143/993/995:** Email services (SMTP, POP3, IMAP)
*   **Port 3306:** MySQL
*   **Port 5038:** Asterisk Call Manager
*   **Port 10000:** Webmin

With this many open ports, it's easy to get distracted. I decided to focus on the web interfaces first, as they typically offer the widest attack surface. 

I navigated to `https://10.10.10.7` in my browser and accepted the self-signed certificate warning.

<!-- screenshot: web application landing page -->

The landing page proudly displayed the Elastix logo and a login prompt. Elastix is an open-source unified communications server that bundles together Asterisk, FreePBX, HylaFAX, Openfire, and a few other tools.

## Initial Foothold
Given the age of the machine (OpenSSH 4.3 indicates an ancient CentOS release), I immediately started searching Exploit-DB and Google for "Elastix exploits." 

I didn't even bother trying default credentials (`admin:admin` or `admin:elastix`) because the search results were overwhelming. Elastix has a notorious history of critical vulnerabilities.

One exploit immediately stood out: a Local File Inclusion (LFI) vulnerability in the `vtigercrm` component, which is bundled with older versions of Elastix (CVE-2012-4869). The vulnerability allows an unauthenticated user to read arbitrary files from the filesystem via the `graph.php` endpoint.

I decided to test this manually using `curl` rather than blindly firing an exploit script. The vulnerable parameter is `current_language`. I crafted a payload to attempt to read `/etc/passwd`. 

Because this is a very old PHP environment, I suspected I might need to use a null byte (`%00`) to terminate the string and prevent the PHP backend from appending `.php` to my requested file path.

```bash
curl -k 'https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/passwd%00&module=Accounts&action'
```

The terminal filled with the contents of the `/etc/passwd` file! The vulnerability was confirmed. 

While reading `/etc/passwd` is great for enumerating users (like `asterisk` and `root`), it doesn't give me a shell. I needed to use this LFI to extract something actionable, like credentials or configuration files.

In an Elastix/Asterisk environment, the holy grail is the FreePBX configuration file, usually located at `/etc/amportal.conf`. This file contains the database passwords and the Asterisk Manager Interface (AMI) credentials in plaintext.

I updated my `curl` payload to target this configuration file.

```bash
curl -s -k 'https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action' | grep -i pass
```

The output revealed a treasure trove of plaintext passwords.

```text
AMPMGRPASS=jEhdIekWmdjE
AMPDBPASS=jEhdIekWmdjE
```

The password `jEhdIekWmdjE` was used for multiple internal administrative accounts.

## Privilege Escalation
At this point, I had several options. I could use the extracted credentials to log into the Elastix web interface, or potentially the Webmin interface on port 10000, and look for a way to execute commands (like a malicious FreePBX module or an authenticated RCE exploit).

However, before going down that rabbit hole, I always test for the simplest, most catastrophic misconfiguration: password reuse on the SSH service. It is shockingly common in older CTF boxes and poorly configured real-world environments for the web administrative password to be the same as the root SSH password.

I fired up SSH and attempted to log in directly as `root`.

```bash
ssh root@10.10.10.7
```

When prompted, I entered the password I extracted from the config file: `jEhdIekWmdjE`.

<!-- screenshot: successful exploitation (reverse shell) -->

The SSH session connected. I ran `id`.

```bash
uid=0(root) gid=0(root) groups=0(root)
```

I was immediately dropped into a root shell. There was no need for lateral movement or further local privilege escalation. I grabbed the `user.txt` and `root.txt` flags.

<!-- screenshot: root flag -->

### Alternative Privilege Escalation (The Intended Path)

While password reuse to root is a valid and often intended shortcut in older boxes, I decided to explore what the *actual* local privilege escalation path was likely intended to be, assuming I had only gained access as the `asterisk` user (perhaps via a different web exploit).

If I had a shell as a low-privileged user, my first step would be to check `sudo` permissions.

```bash
sudo -l
```

<!-- screenshot: sudo -l output -->

The output on this box for a standard user would likely show a massive list of commands allowed without a password, but one specifically stands out:

```text
User asterisk may run the following commands on beep:
    (root) NOPASSWD: /usr/bin/nmap
```

This is a classic GTFOBins vector. Older versions of `nmap` (specifically versions 2.02 to 5.21) included an "interactive" mode designed for debugging and manual testing. This interactive mode allows the user to execute shell commands from within the `nmap` prompt.

Because `sudo` allows the user to execute `nmap` as root, any shell spawned from within `nmap`'s interactive mode will also be executed as root.

```bash
sudo nmap --interactive
```

This drops you into the `nmap>` prompt. From here, you simply prefix a command with `!` to execute it in the shell.

```text
nmap> !sh
# id
uid=0(root) gid=0(root)
```

This alternative path is a fantastic demonstration of why granting `sudo` access to complex binaries without deeply understanding their features can lead to total system compromise.

## Lessons Learned
- **LFI to Configuration Read:** Local File Inclusion is often just the first step. The real value comes from understanding the target application's architecture (like Elastix) and knowing exactly which configuration files (`/etc/amportal.conf`) hold the keys to the kingdom.
- **Password Reuse is Fatal:** Reusing a database or application administrative password for the root SSH account means a single web application flaw results in immediate, total infrastructure compromise. 
- **What tripped me up:** I initially spent a few minutes trying to get a reverse shell via log poisoning with the LFI before realizing I should just check for hardcoded credentials first. Always look for the easy wins before attempting complex exploit chains.
- **Pro Tip:** When you find an LFI, don't just stop at `/etc/passwd`. Build a checklist of common configuration files for popular web frameworks and services to automatically check. 

## References
- Elastix vtigercrm LFI (CVE-2012-4869): https://www.exploit-db.com/exploits/37637
- GTFOBins (Nmap): https://gtfobins.github.io/gtfobins/nmap/#sudo
- Official Hack The Box Page: https://app.hackthebox.com/machines/Beep
