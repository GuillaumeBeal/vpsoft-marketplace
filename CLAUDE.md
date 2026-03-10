# CLAUDE.md — Vpsoft Marketplace

## Projet

Ce dépôt est une marketplace de plugins Claude Code dédiée au consulting VPSoft. Il fournit des skills et commandes pour explorer et diagnostiquer des applications VPSoft via le serveur MCP.

## Structure

- `.claude-plugin/marketplace.json` — Définition de la marketplace et liste des plugins
- `plugins/vpsoft-plugin/` — Plugin principal avec skills et références
- `plugins/vpsoft-plugin/skills/consulting-vpsoft/` — Skill de consulting VPSoft

## Conventions

- Les noms de plugins utilisent le format kebab-case
- Chaque plugin a son propre `.claude-plugin/plugin.json`
- Les skills sont définis dans un dossier `skills/` avec un fichier `SKILL.md`
- Les fichiers de référence sont dans `references/` au sein de chaque skill
- La documentation est rédigée en français

## Stack

- Claude Code plugins / marketplace
- Serveur MCP VPSoft (connexion aux applications VPSoft)
- C# / .NET (côté applicatif VPSoft)
