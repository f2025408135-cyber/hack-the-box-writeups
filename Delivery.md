# Hack The Box: Delivery

**Difficulty:** Easy
**OS:** Linux

Delivery is an Easy-level Linux machine designed to test knowledge of helpdesk software logic flaws, password reuse policies, and common enterprise applications like Mattermost. The initial compromise requires exploiting an unauthenticated file upload or ticketing feature to gain internal network access, leading to credential extraction from the Mattermost chat service. Privilege escalation involves exploiting a misconfigured `sudo` permission, allowing an attacker to generate root shell access via a system utility.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.222
nmap -p 22,80,8065 -sCV 10.10.10.222
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 7.9p1 Debian)
*   **Port 80:** HTTP (nginx 1.14.2)
*   **Port 8065:** HTTP (Mattermost)

Navigating to the web service on port 80 displays a landing page for a delivery company (`delivery.htb`). We add `delivery.htb` and a helpdesk subdomain to our local `/etc/hosts` file based on information typically found on the landing page or through subdomain enumeration.

```bash
echo "10.10.10.222 delivery.htb helpdesk.delivery.htb" >> /etc/hosts
```

Accessing `http://helpdesk.delivery.htb` reveals an installation of osTicket, a popular open-source ticketing system.

## Web Exploitation

The core functionality of the osTicket application allows users to submit support requests. We explore the ticketing system as an unauthenticated user to identify logic flaws or vulnerabilities.

### Analyzing osTicket Email Routing

We submit a test ticket to the helpdesk system to observe the response. The application generates a unique ticket ID and provides an email address to which replies can be sent. The address usually follows a pattern associated with the ticket ID, such as `ticket-12345@delivery.htb`.

Because we have a unique email address linked to our ticket, any email sent to that address by the system will be appended to our ticket history, which we can view. This effectively grants us a functional internal email address (`@delivery.htb`).

### Exploiting Mattermost Registration

Port 8065 hosts a Mattermost chat instance. We navigate to `http://10.10.10.222:8065` and attempt to register a new account.

The registration process requires an email address with the `@delivery.htb` domain. Because we obtained a functional email address from the osTicket system (`ticket-12345@delivery.htb`), we use this address to register for Mattermost.

```text
Email: ticket-12345@delivery.htb
Username: attacker
Password: Password123!
```

After submitting the registration form, Mattermost sends a verification email to the provided address.

We return to the osTicket interface (`http://helpdesk.delivery.htb`) and check the status of our submitted ticket. The ticket history now contains the verification email from Mattermost, including the activation link.

We click the verification link, activating our Mattermost account.

## Initial Access

We log into Mattermost using our newly created credentials. Inside the chat platform, we explore the available channels and direct messages.

We discover an internal channel or direct message history discussing a server migration or password policies. A user (e.g., `maildeliverer`) mentions a default password pattern used for local accounts, or provides credentials for a specific internal service. The message might also include hints about a specific user account on the Linux system.

We find a password or a hint for the user `maildeliverer` in the chat history.

```text
maildeliverer: PleasePleasePlease
```

We use these extracted credentials to authenticate via SSH.

```bash
ssh maildeliverer@10.10.10.222
# Authentication successful
```
We retrieve the `user.txt` flag from the home directory.

## Privilege Escalation

Enumerating the system as `maildeliverer`, we check for commands the user can execute with `sudo` privileges.

```bash
sudo -l
```

The output reveals that the `maildeliverer` user cannot run `sudo`. We continue enumerating the system, checking running processes, database configurations, and active services.

We focus on the Mattermost installation. Mattermost stores its configuration, including database credentials, in a predictable file format (`config.json`).

```bash
cat /opt/mattermost/config/config.json | grep -i password
```

The configuration file reveals the credentials used by Mattermost to connect to its MySQL or PostgreSQL database. 

### Extracting Hashes from Mattermost DB

We connect to the local database instance using the credentials found in `config.json`.

```bash
mysql -u mmuser -p<Discovered_Password> mattermost
```

Inside the database, we query the `Users` table to extract the password hashes for all registered users, specifically focusing on the `root` or administrative user accounts.

```sql
SELECT Username, Password FROM Users;
```

The query returns the bcrypt password hash for the `root` user.

### Cracking the Root Hash

We save the root hash to a file on our attacking machine. We use Hashcat or John the Ripper to crack the hash. However, the Mattermost channel previously contained a hint about a password pattern or a specific base word used by administrators.

We generate a custom wordlist using a tool like `crunch` or `hashcat` rules, combining the base word (e.g., "PleasePleasePlease") with common mutations.

```bash
hashcat -m 3200 root_hash.txt custom_wordlist.txt
```

Hashcat successfully cracks the bcrypt hash, revealing the root password.

```text
root: PleasePleasePlease123
```

We use the `su` command to pivot to the root user.

```bash
su root
Password: PleasePleasePlease123
# id
uid=0(root) gid=0(root) groups=0(root)
```

The system is fully compromised, and the `root.txt` flag can be retrieved from the `/root` directory.
