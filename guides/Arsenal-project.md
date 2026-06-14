---
title: "Arsenal : comment j'ai construit un wiki qui se déploie tout seul"
description: "Retour d'expérience sur la création d'Arsenal - Astro, Docker, Traefik et GitHub Actions - avec le guide complet pour le refaire chez soi."
date: 2026-06-13
tags: ["astro", "docker", "traefik", "ci-cd", "self-hosting", "automatisation", "guide"]
draft: false
---

> L'objectif est de construire un wiki personnel, facile à maintenir, qui se déploie automatiquement à chaque push. (Easy update)

J'ai construit **Arsenal**, le site sur lequel tu lis ces lignes. C'est ma base de connaissances offensive : projets, articles, writeups et ressources.

L'idée directrice tient en une phrase : *j'écris en Markdown, je versionne sur Git, et le site se met à jour tout seul à chaque push.* Aucune intervention manuelle sur le serveur, jamais. Voici comment ça marche, et comment le refaire pour toi.

## La stack, et pourquoi

- **Astro** - génération statique, *content collections* typées, HTML pur. Pour un site de contenu, c'est supérieur à du React : rapide, excellent SEO, zéro JS par défaut.
- **Docker multi-stage** - un stage Node qui build le site, puis une image `nginx:alpine` de quelques Mo qui sert le HTML.
- **Traefik** - reverse proxy + TLS automatique (déjà en place sur mon serveur).
- **Watchtower** - redéploiement automatique quand une nouvelle image arrive.
- **GitHub Actions + GHCR** - la CI qui construit l'image et le registre qui la stocke.

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

1. **Crée les deux repos**: Un pour le contenu et l'autre pour le code de ton application web.
2. **Monte le projet Astro** Créer ton projet Astro avec une DA qui te correspond.
3. **Créer ton docker file** : Exemple de Dockerfile:
```dockerfile
# -- Astro --
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm run ci
COPY . .
RUN npm run build
# -- Nginx --
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```
4. **Rédige le `docker-compose.yml`**: 2 containers → 
      - `arsenal` : ton site, qui tire l'image depuis GHCR.
      - `watchtower` : qui surveille le registre et redéploie le site
5. **Créer deux workflows** : example: `build-and-push.yml` côté code (build de l'image + push GHCR), et `trigger-rebuild.yml` côté contenu (envoie un `repository_dispatch` au repo arsenal à chaque nouveau push de content).
6. **Configure un `DISPATCH_TOKEN`** : Sur le repo `content`, crée un token Github:
      6.1 [Personal access tokens](https://github.com/settings/personal-access-tokens) -> Fine-grained token -> Generate new token
      6.2 Donne lui nom / durée d'expiration / repo `code` en accès / Permissions: Cherche `content` et coche puis change en `read and write`.
      6.3 Ajoute le token dans les secrets du repo `content` (repo -> settings -> secrets & variables -> Actions -> Secrets -> New repo secret) sous le nom `DISPATCH_TOKEN` -> meme nom que dans le workflow `trigger-rebuild.yml`.
      6.4 Rend le package GHCR public dans le repo `code` -> https://github.com/users/USERNAME/packages/container/REPO_NAME/settings 
7. **Configure ton DNS et publie ton site** `docker compose up -d` (téléchargement de l'image depuis GHCR) + Configuration de Traefik pour le reverse proxy et le TLS automatique.

## Au quotidien

Le résultat, c'est exactement ce que je voulais : j'écris un fichier `.md`, je fais `git push`, et environ trois minutes plus tard c'est en ligne. Pas de SSH, pas de build manuel, pas de copie de fichiers. La maintenance des dépendances suit la même logique - je bump une version, je push le repo de code, la CI rebuild.