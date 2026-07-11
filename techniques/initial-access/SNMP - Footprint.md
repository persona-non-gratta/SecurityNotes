---
tags:
  - footprint
  - system
  - network
  - protocol
cssclasses:
  - "[[SNMP|SNMP]]"
---
# Dangerous Settings 

| Settings                               | Description                                                                      |
| -------------------------------------- | -------------------------------------------------------------------------------- |
| rwuser noauth                          | Provides access to the full tree without Auth                                    |
| rwcommunity "communuty string" "IPv4"  | Provides access to the full tree regardless of where the requests were sent from |
| rwcommunity6 "communuty string" "IPv6" | Provides access to the full tree regardless of where the requests were sent from |
`snmpwalk` - query the OIDs with their information
`Onesextyone` - brute-force the names of community strings
`braa` - brute-force individuals OID

```bash
snmpwalk -v2c -c public <ip>
```

```bash
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt <ip>
```

```bash
braa <community string>@<IP>:.1.3.6.* 
braa public@<IP>:.1.3.6.*
```

To filter all useless information:
```bash
grep -i "flag\|pass\|secret\|key\|wrong type\|string" snmp.txt
```

`Wrong Type` = custom data