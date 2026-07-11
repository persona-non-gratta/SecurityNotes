
## Pre-Requiremets
``` bash
ben@kobold:/home$ find / -group "operator" 2>/dev/null   
find / -group "operator" 2>/dev/null
/privatebin-data
/privatebin-data/certs
/privatebin-data/certs/key.pem
/privatebin-data/certs/cert.pem
/privatebin-data/data
/privatebin-data/data/purge_limiter.php
/privatebin-data/data/bd
/privatebin-data/data/bd/b5
/privatebin-data/data/.htaccess
/privatebin-data/data/e3
/privatebin-data/data/traffic_limiter.php
/privatebin-data/data/salt.php
```


### MITM (Man-In-The-Middle Attack) - Certificate Compromission
Type of the attacks when an **attacker** is between server and host, **intercepting all incoming and outgoing traffic.**    Now, let's take a look: we have privatebin's **certification** (`cert.pem`) and **private** key (`key.pem`) both are used for establishing secure connection via HTTPS, previously all the time we have been used `-k` flag in curl and ffuf for skipping **certification's validation check,** because it was **outdated**. Nothing prevents us from creating our own, new and fresh certification and signing it with private key:

```bash
openssl req -new -key key.pem -out request.csr #basic information (name, country.. etc)
openssl x509 -req -in request.csr -signkey key.pem -out cert.pem -days 365 #creating new, self-signed certificate
```

```bash
openssl #linux built-in tool for working with keys, certs, TLS/SSL, encryption and signatures 
req -new #subcomand for working with Certificate Signing Request (CSR) or crecating certificates (x509)
-key #our private key
-out request.csr #command's result

x509 #self-signed certificate command
-req -in request.csr #provided information about cert's owner
-signkey #private key
-out #command's result
-days 365 #certificate validation time
```

Now, we just need to replace old certificate with out new-created one (to make sure, that certificate will work, we can check nginx config): 

``` bash
ssl_certificate     /path/to/cert.pem;
ssl_certificate_key /path/to/key.pem;
```

# Conclusion
With a private key and a new, forged certificate, an attacker can position themselves between the user and the server, intercepting all traffic (which can be done easily and conveniently using Wireshark) to carry out an Evil Twin attack (MITRE ATT&CK: T1557.004) attack, but at the packet level, intercepting account credentials, cookies, session IDs, and clipboard contents
