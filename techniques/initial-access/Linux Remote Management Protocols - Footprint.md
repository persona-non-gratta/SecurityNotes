---
tags:
  - footprint
  - protocol
  - linux
  - system
  - network
---
---
# SSH
#### Dangerous Settings

| Setting                        | Description                                                                                                             |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `PasswordAuthentication yes`   | Allows password-based (could be sniffed, brute-forced)                                                                  |
| `PermitEmpptyPasswords yes`    | Allow the use of empty password (works only if /etc/shadow is clear for user)                                           |
| `PermitRootLogin yes`          | Allows to log in as the root                                                                                            |
| `Protocol 1`                   | Force use of SSH-1 (vulnerable for [[MITM atttack]] - [CVE-2020-14145](https://www.cvedetails.com/cve/CVE-2020-14145/)) |
| `X11Forwarding yes`            | Forwarding GUI Apps - keylogging, espionage                                                                             |
| `AllowTcpForwarding yes`       | ...                                                                                                                     |
| `PermitTunnel`                 | ...                                                                                                                     |
| `DebianBanner yes`             | Information Disclosure                                                                                                  |
| `PreferredAuthentications yes` | allows specifying authentication settings                                                                               |

1. `AllowTcpForwarding yes` - creating forwarded TCP connections:
	- **Local forwarding (`-L`)**: `ssh -L 8080:internal-host:80 user@target` — opens a port on _your_ machine that tunnels to a host _target_ can reach. Useful when target has access to an **internal web app,** DB, etc., that you can't reach directly.
	
	- **Remote forwarding (`-R`)**: `ssh -R 9000:127.0.0.1:22 user@target` — the reverse; exposes a port on the _target_ that maps back to a service on _your_ machine. Used for things like exfiltrating a shell back out of a network with restrictive egress, or giving the target box access to a tool sitting on your attack box. (firewall bypassing - reverse shell)
- https://www.youtube.com/watch?v=N8f5zv9UUMI
	
	- **Dynamic forwarding (`-D`)**: `ssh -D 1080 user@target` — turns the SSH client into a SOCKS proxy. Point `proxychains` at `127.0.0.1:1080` and now _any_ tool (nmap, curl, etc.) routes through the compromised box. This is the one you'll use most in an internal pentest once you land SSH creds — it effectively makes the box your pivot point into whatever network segment it sits on.

2. `PermitTunnel` - useful for #lateral_movement; allows user to create independent tun/tap network interfaces. `tun` - creating layer 3 interface (like we are in the connected via [[VPNs]]) `tap` - layer 2 (like we are plugged in)

## Footprinting 

```python
./ssh-audit.py <ip> # version enumeration
```


Specifying authentication method - `PreferredAuthentications` with brute-force

```bash
 ssh -v cry0l1t3@10.129.14.132
OpenSSH_8.2p1 Ubuntu-4ubuntu0.3, OpenSSL 1.1.1f  31 Mar 2020
debug1: Reading configuration data /etc/ssh/ssh_config 
...SNIP...
debug1: Authentications that can continue: publickey,password,keyboard-interactive
```

```bash
ssh -v cry0l1t3@10.129.14.132 -o PreferredAuthentications=password
```


---

# Rsync
Synchronization tool, operating op port `873 or 31`. Could be used for backups or file transmitting (configured via SSH). Algorithm reduces amount of data due to transmitting only differences / performed changes, while original version already exists.

### Basic Footprint
1. Enumeration
```bash
sudo nmap -sV -p 873 127.0.0.1 
 ...
 PORT STATE SERVICE VERSION 
 873/tcp open rsync (protocol version 31)
```

2. Connecting using ncat
```bash
   nc -nv 127.0.0.1 873                            # no DNS + Verbose
   (UNKNOWN) [127.0.0.1] 873 (rsync) open 
   @RSYNCD: 31.0 
   @RSYNCD: 31.0 
   #list 
   dev Dev Tools 
   @RSYNCD: EXIT 
```

3. Listing 
```bash
rsync -av --list-only rsync://127.0.0.1/dev
```

4. Transmitting (+ using SSH)
```bash
rsync -av rsync://127.0.0.1/dev                # archive mode + verbose
rsync -av rsync://127.0.0.1/dev -e ssh
rsync -av rsync://127.0.0.1/dev -e "ssh, p2200"
```

`archive mode` -  copy directory recursively (everything inside the folders) ; preserves symlinks ; clones  files  exactly

Guide for understanding syntax (over SSH ) https://phoenixnap.com/kb/how-to-rsync-over-ssh

# R-Sevices
`r-services` were the de facto standard for remote access between Unix operating systems until they were replaced by the Secure Shell (`SSH`) protocols and commands due to inherent security flaws built into them.

Span across ports `512`, `513` `514` 

| Command | Daemon  | Port | Description                                                                                                                       | Protocol | Usage                                                                                             |
| ------- | ------- | ---- | --------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------- |
| rcp     | rshd    | 514  | Remote copy from system co system (**can overwrite files without any warning**)                                                   | TCP      |                                                                                                   |
| rsh     | rshd    | 514  | Opens a shell on a remote machine without login procedure. Relies only on trusted entries `/etc/hosts.equiv` and `.rhosts`        | TCP      | was used for non-interactive commands. if command prompt is empty automatically performs `rlogin` |
| rexec   | rexecd  | 512  | Enables remote execution through unecrypted socket and requires user's `login and password`. Authentication ignore trusted files. | TCP      | Password only, useful for automated-scripts                                                       |
| rlogin  | rlogind | 513  | Enables to log in to remote computer. Authentication is overridden by the trusted entries                                         | TCP      | Full interactive (like dor administrating)                                                        |

The primary concern of `r-servies`, and one of the primary reasons `SSH` was introduces to replace it, is the inherent issues regarding access control for these protocols. `r-services` rely of `/etc/hosts.equiv`  and `.rhosts` files on the system. Both files contains `Hostname or IPs` and name of trusted `users`.


> [!NOTE] Details
> `.rhosts` provides access based on usernames, while `/etc/hosts.equiv` file is recognized as the global configuration 

###### `/etc/hosts.equiv `
contains trusted entries to grant full access without any authentication
```bash
cat /etc/hosts.equiv 
# <hostname> <local username>
bubuntu      bubuntuuser
```

###### Sample `.rhosts` File
```bash
cat .rhosts
htb-student     10.0.17.5
+               10.0.17.10 # "+" is like a wildcard, meaning "any", disabling security checks
+               +            
```

#### Log in && listing Users
``` bash
rlogin 10.0.17.2 -l htb-student # log in 

rwho                  # can be broadcasted!
root web01:pts/0 Dec 2 21:34 
htb-student workstn01:tty1 Dec 2 19:57 2:25


rusers -al 10.0.17.5 htb-student 10.0.17.5:console Dec 2 19:57 2:25 # extended version
```