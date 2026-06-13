---
title: "Arsenal : comment j'ai construit un wiki qui se déploie tout seul"
description: "Retour d'expérience sur la création d'Arsenal — Astro, Docker, Traefik et GitHub Actions — avec le guide complet pour le refaire chez soi, pièges inclus."
date: 2026-06-13
tags: ["astro", "docker", "traefik", "ci-cd", "self-hosting", "automatisation", "guide"]
draft: false
---

J'ai construit **Arsenal**, le site sur lequel tu lis ces lignes. C'est ma base de connaissances offensive : projets, articles, writeups et ressources. J'avais déjà un wiki de notes rapides et un portfolio, mais il me manquait un endroit pour recenser proprement ce que je produis, **facilement mis à jour**.

L'idée directrice tient en une phrase : *j'écris en Markdown, je versionne sur Git, et le site se met à jour tout seul à chaque push.* Aucune intervention manuelle sur le serveur, jamais. Voici comment ça marche, et comment le refaire pour toi.

## La stack, et pourquoi

- **Astro** — génération statique, *content collections* typées, HTML pur. Pour un site de contenu, c'est supérieur à du React : rapide, excellent SEO, zéro JS par défaut.
- **Docker multi-stage** — un stage Node qui build le site, puis une image `nginx:alpine` de quelques Mo qui sert le HTML.
- **Traefik** — reverse proxy + TLS automatique (déjà en place sur mon serveur).
- **Watchtower** — redéploiement automatique quand une nouvelle image arrive.
- **GitHub Actions + GHCR** — la CI qui construit l'image et le registre qui la stocke.

## L'architecture : deux repos

C'est le choix central. Je sépare le code du contenu :

- **`arsenal`** *(privé)* — le projet Astro + toute l'infra (Dockerfile, compose, workflow CI).
- **`arsenal-content`** *(public)* — uniquement le Markdown, rangé par catégorie (`articles/`, `writeups/`, `guides/`, `ressources/`, `projets/`, `etudes/`).

Pourquoi le contenu public ? Ça me donne un backup portable de mes notes, la possibilité que des gens proposent des corrections par PR, et de la transparence. Le code source, lui, reste privé. L'image Docker ne contient que le HTML déjà buildé, donc la rendre publique n'expose rien de plus que le site lui-même.

## Le pipeline

Le point clé à comprendre : **mon serveur ne build jamais le site.** Il tire une image déjà cuite. Le build se passe chez GitHub.

```
push .md (arsenal-content)
      │  repository_dispatch
      ▼
build image (arsenal, via GitHub Actions)
      │  docker push
      ▼
GHCR (le registre d'images)
      │  pull
      ▼
Watchtower (sur mon serveur) → recrée le conteneur
      ▼
Traefik → arsenal.louispernet.app
```

## Le refaire chez toi, étape par étape

1. **Crée les deux repos** : `arsenal` (privé), `arsenal-content` (public).
2. **Monte le projet Astro** avec des *content collections* et un schéma de validation (Zod). Chaque fichier Markdown a un frontmatter typé ; si un champ obligatoire manque, le build échoue plutôt que de publier une entrée cassée.
3. **Écris un Dockerfile multi-stage** : build Node, puis `nginx:alpine` qui sert le `dist/`.
4. **Rédige le `docker-compose.yml`** avec les labels Traefik et un Watchtower *scopé* (pour qu'il ne surveille que ce projet, pas tous tes conteneurs).
5. **Mets deux workflows** : `build-and-push` côté code (build + push GHCR), et `trigger-rebuild` côté contenu (envoie un `repository_dispatch` au repo de code à chaque push).
6. **Configure un `DISPATCH_TOKEN`** : un PAT *fine-grained* avec la permission *Contents: write* sur le repo de code, stocké en secret dans le repo de contenu. C'est lui qui autorise le contenu à réveiller le build.
7. **Rends le package GHCR public** après le premier build (sinon ton serveur ne peut pas le tirer sans authentification).
8. **Pointe ton DNS** vers le serveur, puis `docker compose up -d`.

## Au quotidien

Le résultat, c'est exactement ce que je voulais : j'écris un fichier `.md`, je fais `git push`, et environ trois minutes plus tard c'est en ligne. Pas de SSH, pas de build manuel, pas de copie de fichiers. La maintenance des dépendances suit la même logique — je bump une version, je push le repo de code, la CI rebuild.