---
title: "ADCS ESC1 expliqué avec une analogie simple"
description: "Comprendre l'escalade ESC1 sur AD CS sans se noyer dans les OID : l'histoire du formulaire avec un champ de trop."
date: 2026-05-28
tags: ["adcs", "pki", "privesc"]
draft: false
---

## Le problème en une phrase

Un template de certificat **mal configuré** laisse le demandeur choisir
*lui-même* l'identité pour laquelle le certificat est émis. Autrement dit :
tu remplis le formulaire, et tu peux écrire le nom de quelqu'un d'autre.

## L'analogie du formulaire

Imagine un guichet qui imprime des badges. Normalement, ton badge porte
**ton** nom — l'agent le remplit à partir de ta pièce d'identité. ESC1,
c'est un guichet où le champ "nom sur le badge" est laissé **éditable par
toi**. Tu écris "Directeur", et la machine imprime un badge "Directeur"
parfaitement valide, signé par l'autorité.

Le certificat émis est ensuite utilisable pour s'authentifier **en tant que**
cette identité. Champ éditable + authentification = escalade.

## Les conditions réunies

- le template autorise `ENROLLEE_SUPPLIES_SUBJECT`
- l'EKU permet l'authentification client
- un utilisateur peu privilégié a le droit de s'enrôler

Réunis ces trois cases et le chemin vers Domain Admin s'ouvre.

## Remédiation

Retirer `ENROLLEE_SUPPLIES_SUBJECT` des templates d'authentification, ou
exiger une approbation manager. Le champ ne doit jamais être à la main du
demandeur.
