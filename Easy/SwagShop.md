# Hack The Box: SwagShop

**Difficulty:** Easy
**OS:** Linux

SwagShop is an Easy-level Linux machine designed to highlight vulnerabilities in outdated e-commerce platforms, specifically Magento. The initial foothold requires combining directory brute-forcing with an unauthenticated Remote Code Execution (RCE) vulnerability present in older Magento versions. Privilege escalation is achieved by exploiting an insecure `sudo` configuration that allows a low-privileged user to run `vi` as root.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target machine.

```bash
nmap -p- --min-rate 10000 10.10.10.140
nmap -p 22,80 -sCV 10.10.10.140
```

The scan identifies two open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.18)

Navigating to the web service on port 80 displays an online store named "SwagShop." The footer or source code reveals the platform is Magento, a popular open-source e-commerce system written in PHP.

We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster` to identify administrative interfaces or hidden files.

```bash
gobuster dir -u http://10.10.10.140 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals standard Magento paths such as `/app/`, `/downloader/`, and an administrative login portal, often located at a non-standard path like `/index.php/admin` or `/index.php/admin123`. We also discover the version information in files like `/RELEASE_NOTES.txt`.

The Magento version is identified as 1.9.0.0.

## Web Exploitation

Magento 1.x has a history of significant vulnerabilities, including the notorious "Shoplift" bug (CVE-2015-1397). This specific version (1.9.0.0) is known to be vulnerable to unauthenticated remote code execution.

### Exploiting Magento RCE (Shoplift / CVE-2015-1397)

The Shoplift vulnerability involves an SQL injection in the administrative interface that allows an attacker to create a new administrator account. Once authenticated as an administrator, the attacker can leverage file upload features or layout template modifications to achieve RCE.

There are numerous publicly available exploit scripts for this CVE. A common script automates the process of creating the admin account and dropping a PHP web shell.

We download a suitable Python exploit script from Exploit-DB or GitHub and execute it against the target.

```bash
python2 shoplift.py http://10.10.10.140
```

The script successfully executes the SQL injection and creates a new administrative user with a known username and password (e.g., `forme:forme`).

We navigate to the administrative login portal (`http://10.10.10.140/index.php/admin` or similar) and authenticate using the newly created credentials.

## Initial Access

As an authenticated administrator, we explore the Magento backend. We have two primary methods for gaining a shell:

1.  **File Upload (e.g., Magento Connect Manager):** The `/downloader/` path identified during reconnaissance leads to the Magento Connect Manager. We log in using the same administrative credentials. From here, we can package a PHP web shell into a `.tgz` file mimicking a Magento extension and upload it.
2.  **Layout/Template Manipulation:** We can navigate to `System -> Configuration -> Design -> HTML Head` and inject PHP code into the "Miscellaneous Scripts" section, provided the server allows PHP execution within layout files (often disabled by default).

The most reliable method involves exploiting another known feature/vulnerability (CVE-2015-2061), which allows authenticated administrators to achieve RCE via the `froala` editor or by directly uploading malicious packages via the Connect Manager.

We package a reverse shell (`shell.php`) into a `.tgz` archive and upload it through the Magento Connect Manager. Alternatively, we use an exploit script specifically designed to upload the shell post-authentication.

Once the shell is uploaded, we navigate to its location (typically `/errors/shell.php` or `/media/shell.php` depending on the exploit method used).

```text
http://10.10.10.140/media/shell.php?cmd=id
```

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We stabilize our shell and retrieve the `user.txt` flag from the `/home/haris` directory.

## Privilege Escalation

Enumerating the system as `www-data`, we check for commands the user can run using `sudo`.

```bash
sudo -l
```

The output reveals that the `www-data` user can run the `vi` editor as `root` without a password, but only on files within the `/var/www/html/` directory.

```text
Matching Defaults entries for www-data on swagshop:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on swagshop:
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*
```

### Abusing Sudo & vi

The `vi` (or `vim`) text editor has a well-known privilege escalation vector. The editor includes a feature that allows users to execute external shell commands from within the interface. Because `www-data` can execute `vi` as root via `sudo`, any shell command executed from within the editor will also run as root.

While the `sudo` rule restricts us to editing files within `/var/www/html/`, this does not prevent us from executing shell commands once the editor is open.

We execute the permitted `sudo` command on a dummy file or an existing file within the allowed directory.

```bash
sudo /usr/bin/vi /var/www/html/test.txt
```

Once the file opens in the `vi` editor, we enter "Command Mode" by pressing the `Esc` key (or `:` if already in normal mode). We then type the following command to spawn an interactive bash shell:

```text
:!/bin/bash
```

The `!` character instructs `vi` to execute the subsequent text as a shell command. The command executes within the context of the `sudo` process (root), dropping us into a root shell.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
