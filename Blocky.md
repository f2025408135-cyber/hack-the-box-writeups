# Hack The Box: Blocky

**Difficulty:** Easy
**OS:** Linux

Blocky is an Easy-level Linux machine designed to demonstrate common misconfigurations in web application development and the dangers of password reuse. The initial compromise requires identifying exposed development files (specifically Java archive files) on a WordPress site, decompiling them to extract hardcoded credentials, and using those credentials to access a backend system. Privilege escalation relies on exploiting `sudo` privileges assigned to a user account.

---

## Reconnaissance

We begin the assessment with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.10.37
nmap -p 21,22,80,8192 -sCV 10.10.10.37
```

The scan reveals the following open ports:
*   **Port 21:** FTP (ProFTPD 1.3.5a)
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.18)
*   **Port 8192:** HTTP (Macromedia Web Server)

Navigating to port 80 displays a landing page for "Blocky," a WordPress blog focused on Minecraft. We add `blocky.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.10.37 blocky.htb" >> /etc/hosts
```

We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster` to identify application paths and administrative interfaces.

```bash
gobuster dir -u http://10.10.10.37 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals standard WordPress paths (`/wp-admin`, `/wp-content`) and an interesting custom directory: `/plugins`. 

## Web Application Analysis

We access the `/plugins` directory via the web browser. The server has directory listing enabled, revealing two files available for download:
*   `BlockyCore.jar`
*   `grief.jar`

These appear to be compiled Java archives (JAR files), likely custom plugins or backend code related to the Minecraft server implied by the site's theme.

### Decompiling Java Archives

We download `BlockyCore.jar` and `grief.jar` to our attacking machine. JAR files are simply ZIP archives containing compiled Java classes (`.class` files). We can extract them and decompile the bytecode to read the original Java source code.

We use a Java decompiler such as `jd-gui`, `cfr`, or an online service to analyze the contents of `BlockyCore.jar`.

```bash
# Using CFR command-line decompiler
java -jar cfr.jar BlockyCore.jar --outputdir decompiled
```

Reviewing the decompiled source code in `BlockyCore.java`, we locate the database connection logic. The developer hardcoded the MySQL credentials directly into the application code.

```java
// Snippet from decompiled BlockyCore.java
package com.myfirstplugin;

public class BlockyCore {
    public String sqlHost = "localhost";
    public String sqlUser = "root";
    public String sqlPass = "8YsqfCTnvxAUeduzjNSXe22";
    // ...
}
```

The decompilation reveals plaintext database credentials: `root:8YsqfCTnvxAUeduzjNSXe22`.

## Initial Access

We have obtained sensitive credentials, but port 3306 (MySQL) is not exposed externally. However, we have several other services to test these credentials against, including the WordPress login (`/wp-admin`), FTP (port 21), and SSH (port 22).

A common security anti-pattern is password reuse across different services or accounts. We test the extracted database password (`8YsqfCTnvxAUeduzjNSXe22`) against the system users.

From the WordPress site authors or simple enumeration, we identify a primary user named `notch`.

We attempt to authenticate via SSH using these credentials.

```bash
ssh notch@10.10.10.37
# Password: 8YsqfCTnvxAUeduzjNSXe22
# Authentication successful
```
We retrieve the `user.txt` flag from Notch's home directory.

*(Note: Alternatively, we could log into the WordPress dashboard and upload a malicious plugin or modify a theme file to gain a web shell as `www-data`, then pivot to `notch`, but SSH access is more direct).*

## Privilege Escalation

Enumerating the system as `notch`, we check for commands the user can execute with `sudo` privileges.

```bash
sudo -l
```

The output reveals that the user `notch` has extensive administrative rights on the system.

```text
Matching Defaults entries for notch on blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on blocky:
    (ALL : ALL) ALL
```

### Abusing Sudo ALL

The `(ALL : ALL) ALL` configuration indicates that the user `notch` can execute any command as any user (including `root`) by simply prepending `sudo` to the command and providing their own password.

This is a trivial privilege escalation path. We execute a system shell (`su` or `sh`) using `sudo`.

```bash
sudo su -
```

The system prompts for `notch`'s password. We provide the password extracted from the JAR file (`8YsqfCTnvxAUeduzjNSXe22`).

The command executes successfully, granting immediate root access to the system.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
