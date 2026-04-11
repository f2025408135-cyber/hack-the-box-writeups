# Hack The Box: Cereal

**Difficulty:** Hard
**OS:** Windows

Cereal is an advanced Windows machine focused heavily on .NET code analysis, insecure deserialization, Cross-Site Scripting (XSS), and bypassing IP restrictions. The exploitation path requires piecing together information from an exposed Git repository, forging a JWT to access a restricted API, and chaining an XSS vulnerability with an insecure deserialization flaw to obtain a shell. Privilege escalation involves abusing a misconfigured SQLite database.

---

## Reconnaissance

We start by performing a port scan on the target machine.

```bash
nmap -p- --min-rate 10000 10.10.10.217
nmap -p 22,80,443 -sCV 10.10.10.217
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH)
*   **Port 80:** HTTP
*   **Port 443:** HTTPS

Browsing to port 80/443 reveals a web application for cereal enthusiasts. By examining error messages on specific subdomains like `source.cereal.htb`, we are able to leak absolute paths.

Enumerating directories reveals the presence of a `.git` repository, though standard directory lists like `DirectoryList` might miss it if `.git` is hidden or forbidden without specific enumeration tools like `Raft` wordlists.

```bash
# Example tool for grabbing the repository
git-dumper http://source.cereal.htb/.git /local/path/cereal-git
```

## Source Code Analysis

With the source code downloaded, we analyze the application's backend logic, written in .NET.

Reviewing the Git commit history provides deep insight into the application's evolution, notably:
1.  **Deserialization:** We identify the endpoints and functions handling serialized data. The application utilizes a custom deserialization routine.
2.  **Authentication:** The source code reveals a hardcoded JWT (JSON Web Token) secret used for generating and validating API tokens.

Using the hardcoded JWT secret, we can forge our own authentication tokens using a custom script or a tool like `jwt.io`.

## Web Exploitation

We attempt to use the forged JWT to access authenticated pages. Examining the frontend React JavaScript indicates that the token is expected to be stored in the browser's local storage. However, the browser frequently clears the storage, requiring us to intercept requests using Burp Suite and manually inject our forged JWT header to interact with the API.

### Deserialization and Bypasses

The application implements a `BadWords` filter to prevent standard `ysoserial.net` payloads from executing during the deserialization process. This necessitates the creation of a custom deserialization payload.

By identifying the specific JSON library in use (`Newtonsoft.Json`), we can research known exploitation vectors. Our custom payload needs to bypass the blacklist while still achieving code execution.

### Chaining XSS and Deserialization

Further examination of the frontend JavaScript reveals additional administrative routes. Using `npm audit` against the application's package dependencies uncovers an outdated version of `react-marked-markdown` which is vulnerable to Cross-Site Scripting (XSS). This vulnerability is located on the `/admin` endpoint.

We test the XSS vulnerability with a simple payload to confirm execution.

However, the deserialization endpoint restricts requests based on IP address, preventing direct external exploitation. To bypass this, we chain the vulnerabilities:
1. We craft an XSS payload designed to be executed by an internal administrator or browser bot.
2. This XSS payload uses `XMLHttpRequest` to forge a request *from the internal context* to the vulnerable deserialization endpoint, bypassing the IP restriction.

### Gaining Initial Access

The exploitation chain requires encoding the payload to avoid bad character filtering (using Base64). 

The full exploit script performs the following:
1. Forges a valid JWT.
2. Constructs the custom .NET deserialization payload designed to download an ASPX web shell.
3. Wraps the deserialization request inside an XSS payload using `XMLHttpRequest`.
4. Submits the XSS payload to the `/admin` endpoint.

When the XSS is triggered internally, the deserialization payload executes, downloading our ASPX web shell to the server. Accessing the web shell grants initial code execution as the IIS service account.

## Privilege Escalation

Enumerating the system from our initial shell reveals an interesting SQLite database file. Using our web shell or an established reverse shell, we download the database for local analysis.

```powershell
# Example PowerShell one-liner to convert a file to base64 for exfiltration
[convert]::ToBase64String((Get-Content -path "C:\path\to\database.sqlite" -Encoding byte))
```

Analyzing the SQLite database offline uncovers sensitive administrative credentials or tokens that can be used to elevate privileges to `NT AUTHORITY\SYSTEM` or a high-privileged administrative user, leading to full compromise of the machine.
