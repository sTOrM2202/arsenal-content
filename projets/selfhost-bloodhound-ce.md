---
title: "Self-hosting BloodHound CE 8 avec bloodhound-cli"
description: "Déployer une instance BloodHound Community Edition en local via Docker et bloodhound-cli, sans dépendre du SaaS."
date: 2026-05-11
tags: ["bloodhound", "docker", "infra"]
draft: false
---

## L'objectif

Avoir **mon** BloodHound CE qui tourne en local : pas de données qui partent
ailleurs, et une stack reproductible sur le serveur.

## La stack

`bloodhound-cli` orchestre Postgres + Neo4j + l'API + l'UI dans des
conteneurs Docker. Une commande pour tout lancer.

```bash
# installation
curl -L https://ghst.ly/getbhce | docker compose -f - up -d

# ou via bloodhound-cli
./bloodhound-cli install
./bloodhound-cli containers start
```

## Premier login

Le mot de passe admin initial est généré au premier démarrage et affiché
dans les logs — à changer immédiatement.

```bash
docker compose logs bloodhound | grep -i password
```

## Ingestion

Les collecteurs (SharpHound / azurehound) produisent des ZIP qu'on uploade
directement dans l'UI. À partir de là, les requêtes Cypher et les chemins
pré-câblés font le reste.
