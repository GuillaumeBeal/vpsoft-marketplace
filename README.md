# Vpsoft Marketplace

Marketplace de plugins Claude Code pour le consulting autour de l'application VPSoft.

## Plugins disponibles

| Plugin | Version | Description |
|--------|---------|-------------|
| **vpsoft-plugin** | 1.1.0 | Outils et skills pour explorer, comprendre et configurer une application VPSoft via le serveur MCP. Exploration du modèle de données, code dynamique, audit/diagnostic, et configuration (tables/champs, menus, filtres, indicateurs, dashboards, workflows, pages dynamiques, business rules). |

### Skills inclus

| Skill | Description |
|-------|-------------|
| **consulting-vpsoft** | Exploration et compréhension d'une application VPSoft via le serveur MCP. 13 workflows couvrant : découverte de l'application, modèle de données, code dynamique, requêtage, diagnostic/audit, code source C#, diagrammes Mermaid, module Form (questionnaires, campagnes, import/export), permissions par rôle. |
| **spec-orchestrator** | Orchestrateur de spécifications fonctionnelles. Lance des agents autonomes en parallèle pour documenter les vues listes et formulaires VPSoft de chaque entité, évitant le dépassement de la fenêtre de contexte. |
| **vpsoft-config** | Configuration de VPSoft via les DynamicFunctions MCP : tables/champs (expression C#, formule, reverse-link, unités, arbres), visibilité/labels/rendu, menus (icônes, ordre, page→menu), filtres de liste, import/export, indicateurs & dashboards, workflows & transitions, visuels de liste + quickfilters cliquables, pages dynamiques interactives (cockpit CRUD), business rules no-code, confidentialité, PDF, alertes, aides. Inclut le code complet et recréable de ~60 fonctions `Mcp*`. |

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
│   └── marketplace.json            # Définition de la marketplace
├── plugins/
│   └── vpsoft-plugin/
│       ├── .claude-plugin/
│       │   └── plugin.json         # Manifest du plugin (v1.1.0)
│       ├── .mcp.json               # Configuration MCP VPSoft
│       └── skills/
│           ├── consulting-vpsoft/
│           │   ├── SKILL.md        # Skill de consulting (13 workflows)
│           │   └── references/     # Documentation de référence
│           ├── spec-orchestrator/
│           │   ├── SKILL.md        # Skill d'orchestration par agents
│           │   └── references/     # Template agent et prompts
│           └── vpsoft-config/
│               ├── SKILL.md        # Skill de configuration (FormBuilder, pages, business rules)
│               └── references/     # 7 docs de référence (code complet des fonctions Mcp*)
└── README.md
```

## Contribuer

Proposez des améliorations via une PR dans ce dépôt.
