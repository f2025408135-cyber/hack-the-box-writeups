---
# Hack The Box: Blocky

| Attribute | Value |
|-----------|-------|
| **Difficulty** | Easy |
| **OS** | Linux |
| **IP** | 10.10.10.37 |
| **Points** | 20 |
| **Release Date** | July 2017 |
| **Retired Date** | December 2017 |
| **My Rating** | 4/10 |

## TL;DR
Enumerated a Minecraft-themed WordPress blog and found an exposed `/plugins` directory with directory listing enabled. Downloaded two Java Archive (JAR) files, decompiled them, and found hardcoded MySQL credentials. Reused those exact credentials to SSH into the box as the user `notch`. Checked `sudo -l` and found `notch` could run any command as any user without a password, leading to a trivial `sudo su` for root.

## Reconnaissance
I started with a quick nmap scan to see what ports were open on this machine.

```bash
nmap -p- -T4 --min-rate 1000 10.10.10.37 -oA nmap/initial
```
<!-- screenshot: nmap initial scan results -->

The initial scan returned a few interesting ports, so I followed up with a detailed service scan on the discovered ports.

```bash
nmap -sC -sV -p 21,22,80,8192 10.10.10.37 -oA nmap/detailed
```

The output was pretty clear:
*   **Port 21:** FTP (ProFTPD 1.3.5a)
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.18)
*   **Port 8192:** HTTP (Macromedia Web Server / phpMyAdmin)

Anonymous FTP login failed, so I moved straight to the web server on port 80. Navigating to `http://10.10.10.37` in my browser brought up a WordPress blog called "BlockyCraft." The theme was heavily focused on Minecraft.

<!-- screenshot: web application landing page -->

I added `blocky.htb` to my `/etc/hosts` file just in case there was any virtual host routing going on, though it didn't seem strictly necessary here.

Since it was WordPress, my immediate thought was to run `wpscan` to enumerate users, themes, and vulnerable plugins.

```bash
wpscan --url http://10.10.10.37 --enumerate u,vp
```

`wpscan` identified a user named `notch` (a nod to the creator of Minecraft), but no immediately obvious vulnerable plugins stood out. I decided to run a standard directory brute-force with `gobuster` to see if there was anything hidden outside the standard WordPress structure.

```bash
gobuster dir -u http://10.10.10.37 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The enumeration quickly found the usual `/wp-admin`, `/wp-content`, and `/wp-includes`, but it also found `/plugins`, `/wiki`, and `/phpmyadmin`.

## Initial Foothold
The `/plugins` directory was the most intriguing non-standard path. I navigated to `http://10.10.10.37/plugins` in my browser.

To my surprise, the server had directory listing enabled (Options +Indexes in Apache). The directory wasn't empty; it contained two files: `BlockyCore.jar` and `grief.jar`.

Finding compiled Java Archives (JARs) sitting in a web directory is a massive red flag. These files were likely custom plugins developed for the Minecraft server that the blog was associated with. I downloaded both files to my attacking machine for analysis.

```bash
wget http://10.10.10.37/plugins/BlockyCore.jar
wget http://10.10.10.37/plugins/grief.jar
```

A JAR file is essentially just a ZIP archive containing compiled Java `.class` files, along with some metadata. While I could unzip them to see the structure, reading the compiled bytecode directly is tedious. I needed a decompiler to convert the `.class` files back into readable `.java` source code.

I used `jd-gui` (Java Decompiler GUI) because it's quick and visual, but command-line tools like CFR or Procyon work just as well. I opened `BlockyCore.jar` in `jd-gui`.

The structure was simple. Inside the `com.myfirstplugin` package, there was a class named `BlockyCore`. I examined the decompiled source code.

```java
package com.myfirstplugin;

public class BlockyCore {
    public String sqlHost = "localhost";
    public String sqlUser = "root";
    public String sqlPass = "8YsqfCTnvxAUeduzjNSXe22";

    public void onServerStart() {
        // ... connection logic using the hardcoded credentials ...
    }
}
```

The developer had hardcoded the MySQL database credentials directly into the plugin's source code. 

*   **Username:** `root`
*   **Password:** `8YsqfCTnvxAUeduzjNSXe22`

At this point, I had database credentials. I knew from my initial nmap scan that MySQL wasn't exposed externally (port 3306 was filtered/closed), but `phpMyAdmin` was running on port 8192 (or `/phpmyadmin` on port 80). I could use these credentials to log into `phpMyAdmin`, modify the WordPress `wp_users` table, change the admin password, log into the WordPress dashboard, and upload a malicious plugin to get a reverse shell as `www-data`.

But that felt like the long way around. A very common flaw in systems (and CTFs) is password reuse. I already knew there was a system user named `notch` from my `wpscan` enumeration and the blog posts.

I decided to try SSHing directly into the box using the username `notch` and the newly discovered database password.

```bash
ssh notch@10.10.10.37
```

It prompted for a password. I pasted in `8YsqfCTnvxAUeduzjNSXe22`.

<!-- screenshot: successful exploitation (reverse shell) -->

The SSH session connected immediately! I was in as the user `notch`. I didn't even need to bother with a reverse shell or navigating the WordPress backend. I grabbed the `user.txt` flag from Notch's home directory.

## Privilege Escalation
With a solid SSH foothold as a standard user, I started my privilege escalation checks. The absolute first thing I check on any Linux machine is what commands the current user is allowed to run with elevated privileges via `sudo`.

```bash
sudo -l
```

<!-- screenshot: sudo -l output -->

The system prompted me for `notch`'s password again. I pasted `8YsqfCTnvxAUeduzjNSXe22`. The output was the holy grail of privilege escalation:

```text
Matching Defaults entries for notch on blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on blocky:
    (ALL : ALL) ALL
```

This configuration meant that the user `notch` was allowed to run *any* command, as *any* user (including root), provided they entered their own password. This is effectively full administrative access to the system.

The privilege escalation couldn't have been simpler. I just needed to use `sudo` to spawn a root shell.

```bash
sudo su -
```
*(Alternatively, `sudo /bin/bash` or `sudo -i` would work just as well).*

The system asked for my password one last time. I provided it, and my prompt changed from `$ ` to `# `.

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

<!-- screenshot: root flag -->

And that's a box. The privesc on this one was incredibly straightforward, almost trivially so. The main lesson here was thorough enumeration and always checking the simplest path (password reuse) before diving into complex exploitation chains.

## Lessons Learned
- **Exposed Development Files:** Leaving compiled binaries or source code (like JAR files) in a publicly accessible web directory with directory listing enabled is a critical security failure. Web directories must be properly configured to deny indexing (`Options -Indexes` in Apache) and restrict access to non-essential files.
- **Hardcoded Credentials:** Embedding passwords directly into source code is never secure. If an attacker gains access to the compiled binary, decompiling it to retrieve the plaintext strings is trivial. Credentials should always be managed via secure environment variables, configuration files stored outside the web root, or dedicated secrets management solutions.
- **Password Reuse:** Using the MySQL root password as the system SSH password for a standard user account completely undermines the principle of least privilege. If one service is compromised (in this case, the database credentials leaked via the JAR), the entire system falls.
- **Overly Permissive Sudo Rights:** Granting a user `(ALL : ALL) ALL` in the sudoers file effectively makes them a root equivalent. If that user's account is compromised, the attacker immediately gains full control of the server. 
- **What tripped me up:** I almost went down the rabbit hole of trying to exploit phpMyAdmin to modify the WordPress database. While that would have worked to get a shell as `www-data`, it would have been significantly more time-consuming than just trying the password over SSH first.
- **Pro Tip:** Whenever you find a password, *always* try it against every known user account on every exposed service (SSH, FTP, web portals) before trying anything complicated. Password reuse is the attacker's best friend.

## References
- JD-GUI Java Decompiler: http://java-decompiler.github.io/
- CFR Java Decompiler: https://www.benf.org/other/cfr/
- Official Hack The Box Page: https://app.hackthebox.com/machines/Blocky
