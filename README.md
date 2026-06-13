# arsenal-content

Contenu (Markdown) du wiki **ARSENAL** — https://arsenal.louispernet.app

Ce repo est **public** : il sert de sauvegarde portable de mes notes et
permet de proposer des corrections par Pull Request.

## Comment ca marche

1. J'ajoute / modifie un fichier `.md` dans le bon dossier
2. `git push` sur `main`
3. Le workflow envoie un signal au repo `arsenal` (prive)
4. `arsenal` rebuild le site, pousse l'image sur GHCR, le serveur la tire

## Arborescence

| Dossier        | Contenu                                            |
|----------------|----------------------------------------------------|
| `articles/`    | Reflexions, analyses techniques                    |
| `writeups/`    | Resolutions HTB, labs AD, CTF                       |
| `guides/`      | Methodologies reutilisables                         |
| `ressources/`  | Outils, cheatsheets, liens                          |
| `projets/`     | Labs, scripts, infra, side-projects                 |
| `etudes/`      | Parcours, certifications                            |

## Frontmatter attendu

Chaque fichier commence par un bloc YAML :

```yaml
---
title: "Titre de la note"
description: "Resume court (optionnel, utile pour le SEO)"
date: 2026-06-10
tags: ["active-directory", "kerberos"]
draft: false   # true = non publie
---
```

Si un champ obligatoire manque, le build echoue (validation par schema).
Champs obligatoires : `title`, `date`. Les autres ont une valeur par defaut.

## Secret a configurer

Dans **Settings > Secrets and variables > Actions** de CE repo :

- `DISPATCH_TOKEN` : un Personal Access Token (fine-grained) ayant la
  permission **Contents: Read and write** sur le repo `arsenal`.
