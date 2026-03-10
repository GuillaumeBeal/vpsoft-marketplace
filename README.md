# Vpsoft Marketplace

Marketplace de plugins Claude Code pour le consulting autour de l'application VPSoft.

## Plugins disponibles

| Plugin | Description |
|--------|-------------|
| **vpsoft-plugin** | Outils et skills pour explorer et comprendre une application VPSoft via le serveur MCP. Exploration du modèle de données, code dynamique, audit et diagnostic. |

## Installation

Ajoutez cette marketplace dans Claude Code :

```
/plugin marketplace add GuillaumeBeal/vpsoft-marketplace
```

Puis installez le plugin :

```
/plugin install vpsoft-plugin@Vpsoft-Marketplace
```

> **Note** : Ce dépôt est privé. Assurez-vous que la variable d'environnement `GITHUB_TOKEN` est définie avec un token ayant accès au repo.

## Structure

```
.
├── .claude-plugin/
│   └── marketplace.json       # Définition de la marketplace
├── plugins/
│   └── vpsoft-plugin/
│       ├── .claude-plugin/
│       │   └── plugin.json    # Manifest du plugin
│       └── skills/
│           └── consulting-vpsoft/
│               ├── SKILL.md   # Définition du skill
│               └── references/  # Documentation de référence
└── README.md
```

## Contribuer

Proposez des améliorations via une PR dans ce dépôt.
