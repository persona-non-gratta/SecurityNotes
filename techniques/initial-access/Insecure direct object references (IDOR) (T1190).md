# Insecure direct object references
**IDOR** - type of access control vulnerabilities **that arises when an application uses user-supplied input to access objects directly.**
### IDOR - Direct reference to static files
IDOR vulnerabilities often arise when **sensitive resources are located in static files on the server-side filesystem.** 

`https://insecure-website.com/static/12144.txt`

In this situation, an attacker can simply modify the filename to retrieve a transcript created by another user and potentially obtain user credentials and otvulnerability with her sensitive data.

### IDOR - direct reference to database objects
`https://insecure-website.com/customer_account?customer_number=132355`
Attacker can alter value of "customer_number" bypassing access controls to review records of other customers or many others posibilities. 

**Arise when information is directly retrieving by information from back-end database**
