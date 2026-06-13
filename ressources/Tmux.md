---
title: "Ma cheatsheet Tmux"
description: "Les commandes pour la mise en place d'un pivot avec Tmux"
date: 2026-02-26
tags: ["cheatsheet", "tmux", "terminal", "optimization", "rapidité", "productivité"]
draft: false
---
# Configuration fichier Tmux (~/.tmux.conf)
1. Fichier de configuration pour Tmux
```bash
# ~/.tmux.conf
set-option -g default-shell /bin/zsh
set -g mouse on
```
2. Relancer la configuration de Tmux
```bash
tmux source-file ~/.tmux.conf
```
## Volets dans une fenêtre
### Créer et supprimer
```bash
Ctrl + B -> Shift + % #Split Vertical
Ctrl + B -> "  #Split horizontal
Ctrl + B -> x  ou Ctrl + D  #ferme le volet actuel 
```
### Changer de Volet
```bash
Ctrl + B -> FLECHE
```
### Basculer le volet en cours (de vertical à horizontal et inversement)
```bash
Ctrl + B -> ESPACE
```
### Resize le volet
```bash
Ctrl + B -> Ctrl + FLECHE
```
# Fenêtre dans une session
### Créer une fenêtre
```bash
Ctrl + B -> C
```
### Renommer la fenêtre en cours
```bash
Ctrl + B -> ,
```
### Fermer la fenêtre en cours
```bash
Ctrl + B -> &
```
### Lister les fenêtres
```bash
Ctrl + B -> w
```
### Changer de fenêtre
```bash
Ctrl + B -> 0-9
```
# Sessions Tmux
### Create Session
```bash
Ctrl + B -> :new
```
### Rename session
```bash
Ctrl + B -> $
```
### Detach from session
```bash
Ctrl + B -> d
```
### Show all sessions
```bash
Ctrl + B -> s
```
### Preview des sessions et fenêtre
```bash
Ctrl + B -> w
```
### Changer de sessions
```bash
Ctrl + B -> (
Ctrl + B -> )
```