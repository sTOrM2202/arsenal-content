---
title: "Quick install ZSH"
description: "Les commandes pour l'installation de ZSH + OhMyZsh"
date: 2026-07-07
tags: ["cheatsheet", "tooling", "zsh", "shell", "ssh"]
draft: false
---

![OMZ](./images/omz.png)

```bash
# Optionnel
sudo apt install curl
# 1. Install ZSH
sudo apt install zsh
# 2. Install OMZ
sudo apt install git
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# 3. Set as default shell for current user
sudo chsh -s $(which zsh) $USER
```
Choose your Zsh [theme](https://github.com/ohmyzsh/ohmyzsh/wiki/themes)
