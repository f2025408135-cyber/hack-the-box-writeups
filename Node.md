# Hack The Box: Node

**Difficulty:** Medium
**OS:** Linux

Node is a Medium-difficulty Linux machine that highlights vulnerabilities specific to the Node.js and MongoDB ecosystem. The machine requires an understanding of NoSQL injection and improper access controls to gain initial access. Privilege escalation requires identifying sensitive data within memory dumps and local file enumeration.

---

## Reconnaissance

The assessment starts with an Nmap scan to identify available services.

```bash
nmap -sC -sV -oA Node 10.10.10.58
```

The scan identifies the following services:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 3000:** Node.js Web Application

Navigating to port 3000 in a web browser displays a simple web page. Enumerating the application's functionality and reviewing the page's source code reveals several API endpoints, notably `/api/users`.

### API Enumeration & Credential Access

Accessing the `/api/users` endpoint returns a JSON response containing usernames and their corresponding password hashes.

```json
{
  "users": [
    {"username": "tom", "password": "hash_here"},
    {"username": "mark", "password": "hash_here"},
    {"username": "myP14ceAdminAcc0unT", "password": "hash_here"}
  ]
}
```

The hashes are extracted and analyzed. The hash for the administrative account, `myP14ceAdminAcc0unT`, is successfully cracked using a tool such as Hashcat or John the Ripper, revealing the plaintext password.

## Web Exploitation & Initial Access

With valid administrative credentials, authentication to the web portal is achieved. The administrative interface provides a feature to download a system backup.

The downloaded file appears to be a Base64-encoded string. Decoding the string reveals it is a password-protected ZIP archive.

### Extracting the Backup

A dictionary attack is launched against the ZIP archive to recover the password.

```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt backup.zip
```

The password is successfully recovered, allowing the extraction of the archive's contents. The archive contains the source code for the Node.js application, including `app.js`.

### Source Code Analysis

Reviewing the application's source code reveals its backend infrastructure. The application utilizes MongoDB for data storage and relies on a specific framework for its administrative features.

The code contains hardcoded credentials for a local user and reveals the structure of the MongoDB database. 

```javascript
// snippet from app.js showing DB connection
const mongoose = require('mongoose');
mongoose.connect('mongodb://mark:password@localhost/node');
```

By connecting via SSH using the credentials recovered either from the cracked hashes or the source code, initial access to the system is obtained as the user `mark`.

```bash
ssh mark@10.10.10.58
# Authentication successful
```
The `user.txt` flag can now be read.

## Privilege Escalation

Initial enumeration of the system as the user `mark` reveals several non-standard files and directories. Among these is a backup directory containing a memory dump or core dump file belonging to a process previously run by `root` or another privileged user.

### Memory Dump Analysis

Analyzing the core dump file using strings analysis and pattern matching reveals sensitive information. 

```bash
strings /var/backups/core_dump | grep -i password
```

The analysis uncovers credentials or an authentication token belonging to the user `tom`.

### Pivoting and Root Access

Using the discovered credentials, a lateral movement pivot is performed to the user `tom`.

```bash
su tom
```

Further enumeration as `tom` identifies a custom binary with the SUID bit set, owned by root. The binary is designed to execute specific administrative tasks but lacks proper input sanitization.

Executing the SUID binary with manipulated arguments or environment variables allows for the execution of arbitrary commands with root privileges.

```bash
# Example exploitation of SUID binary
./custom_suid_binary $(sh)
```

The command execution results in a root shell.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```
The machine is fully compromised and the `root.txt` flag is obtained.
