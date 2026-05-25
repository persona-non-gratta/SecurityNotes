---
tags:
  - initial_access
  - linux
---

#### Impacket - Packet Building (Example from [Archetype CTF - HACK THE BOX - WRITE-UP](<../Archetype CTF - HACK THE BOX - WRITE-UP>) (MSSQL))
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
