---
tags:
  - Methodology
  - enumeration
  - recon
  - nmap
---
# Reconnaissance & Target Discovery
Target: IP ranges, domains, subnets
### Passive Enumeration
1. [[Passive Enumeration Guide]] - OSINT - (DNS, WHOIS, Shodan, certificates)
2.  Public Data Leaks (GitHub; Google Dorks ...) 
3. [[DNS - Footprint]]

### Active Enumeration 
1. **Discover all active hosts, IP ranges, subnet(s). Document all!**
	[[Nmap - Host Discovery]]
	TCP/UDP host discovery(masscan / nmap -sn)
	ARP-scanning (if you are in the same subnet)
		Document all

2. **For each active host, scan ALL TCP / UDP ports**
	[[Nmap - Port Scanning]] ( + `-sU` for UDP)
	[[Nmap - Script Table]] ( check script table and choose)
		Document all!  (Services and Versions)

**3. Identify OS and Versions of running Services**
	Nmap Scan Information (extract data)
	Netcat Banner Grabbing  (`nc <ip> <port>`) - manual identification
		Document all! (Services and Versions)

**4. For each service, do individual enumeration to look for more information and vulnerabilities**
	 Search in Metasploit for service exploits (based on version) 
	 Search in ExploitDB for service exploits (based on version) (`search-sploit`)
	 Look for OS version exploits
	 Nmap Script output (like anonymous login)
	 Google: "Service Version Exploit GitHub"
		 Document all Discovered Vulnerabilities!

**5. Examines File Share - Protocols**
	[[FTP - Footprint]]
	[[SMB - Footprint]]
	[[NFS - Footprint]] (showmount -e ip)
		 Useful: [[NetExec]] 
Look for credentials - users - groups - permissions - Document all Findings!

6.  Check Management Protocols 
	[[IPMI - Footprint]]
	[[SNMP]] (onesixtyone -> snmpwalk) (UDP!)
	[[Oracle TNS - Footprint]]

---
