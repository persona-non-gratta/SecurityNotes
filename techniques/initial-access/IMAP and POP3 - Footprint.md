---
tags:
  - footprint
  - enumeration
  - recon
  - exploitation
  - protocol
cssclasses:
  - "[[SMTP|SMTP]]"
  - "[[IMAP & POP3]]"
---
---
# Dangerous Settings 

| Setting                   | Description                                                    |
| ------------------------- | -------------------------------------------------------------- |
| `auth_debug`              | force server to log all authentication attempts (in plaintext) |
| `auth_debug_passwords`    | submitted password gets logged                                 |
| `auth_verbose`            | Logs unsuccessful auth. attempts                               |
| `auth_verbose_passwrods`  | force server to log all submitted passwords                    |
| `auth_anonymous_username` | specifies the ANONYMOUS username                               |

---
# Footprintring 
```bash
sudo nmap <ip> -sV -p110,143,993,995 -sC
```

```bash
curl -k 'imaps://<ip>' --user user:password # list all mailboxes
```

```bash
openssl s_client -connect <ip>:pop3s
openssl s_client -connect <ip>:imaps
```

https://www.atmail.com/blog/imap-commands/?source=post_page-----5e5c99547f8a---------------------------------------