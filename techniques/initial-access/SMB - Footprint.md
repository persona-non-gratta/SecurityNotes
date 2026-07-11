---
cssclasses:
  - "[[NetExec]]"
  - "[[Impacket - Packet Construction]]"
  - "[[SMB]]"
tags:
  - enumeration
  - lateral_movement
  - exploitation
  - footprint
  - protocol
  - tool
  - recon
---
---
### SMB CLIENT:
The Server Message Blocks (SMB service) allows anonymous access via guest authentication (through a null session):
```bash 
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
SNIP
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user  
```

[**Server Message Block** ](https://en.wikipedia.org/wiki/Server_Message_Block)(by default on port 445) - operates on Application Layer of TCP/IP stack. Protocol is used for share files, printers, serial ports and different communication
between nodes on a network on Windows-Based Machines.

**What means  "anonymous access via guest authentication"? Why is it vulnerable?**
Guest Access means that the server allows **connection without valid credential** (without any password or login), typically mapping  the session (named null session) to the **Guest account.** **Allowing this kind of sessions is the typical misconfiguration of SMB service.**

## SMB Client - Initial Access & Cheat Sheet
SMB allows connection within null-session (in guest/anonymous mode). Using  `smbclient` with flags `-N` (for skipping authentication and enter as a guest) and `-L` (for list all available shares), we can see next output: 

```shell
smbclient -NL <ip address>                        
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backups         Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available

## Some useful commands 
smb:\> ls                  # list files
smb:\> dir                 # same as ls
smb:\> cd folder           # change remote directory
smb:\> pwd                 # show current remote path
smb:\> lcd /dir            # change LOCAL directory

smb:\> get file.txt                # download one file
smb:\> mget *.txt                  # download multiple files
smb:\> put localfile.txt           # upload one file
smb:\> mput *.txt                  # upload multiple files
smb:\> reget file.txt              # resume download

smb:\> mkdir newdir                # create new directory
smb:\> rmdir olddir                # remove any directory
smb:\> rm file.txt                 # delete file
smb:\> rename old.txt new.txt      # rename file

smb:\> allinfo file.txt            # display all info
smb:\> stat file.txt               # persmission check

#downloads the whole directory
smb:\> recurse ON                  
smb:\> prompt OFF
smb:\> mget *

```

``` shell
smbclient -NL <ip address>  # null-session (if allowed)
smbclient -U <username(%password)> <ip address>  # default session
```

``` shell
smbclient -NL <ip address> -c "one of the commands above" #one-line execution
```
---
# [[NetExec]]
# [[Impacket - Packet Construction]]

---
# smbmap

```python 
smbmap -H 192.168.122.198   
smbmap -H 192.168.122.198   -u <USERNAME> -p <PASSWORD> 
smbmap -H 192.168.122.198   -s <SHARE>
smbmap -H 192.168.122.198   --download /path/to/file
smbmap -H -h   
```
---
# rpcclient
```bash
rpcclient -U "" <target>     
```

#### Enumeration commands
``` bash
srvinfo                      # server information
enumdomains                  # enumerate ALL domains deployed in the network
enumdomusers                 # enumerate all users
querydominfo                 # server, user, domain info deployed on the target
queryuser <RID>              # displays all information about selected user
querygroup                   # all info about selected group
netsharegetinfo <share>      # info about specific share
```

Automated user enumeration (bruteforcing)
```bash
for i in $(seq 500 1100);do rpcclient -N -U "" 10.129.14.128 -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
```

Tool Used for directly mounting SMB shares to the selected path and working with them. 
 ``` bash
 sudo mount -t cifs -o username=<username>,password=<password> //ipoftarget/"folder" /path/to/attackers/folder
 ```
---

# enum4linux (noisy)
```bash
./enum4linux-ng.py 192.168.122.198 -A      
```



