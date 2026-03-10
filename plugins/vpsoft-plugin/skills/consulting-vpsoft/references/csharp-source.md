# Cartographie du code source C# — VPSoft

Reference pour explorer le code source C# du socle VPSoft.
Toute l'information sur les projets, controllers et modeles est embarquee ci-dessous.

## Projets principaux

| Projet | Role | Chemins cles |
|--------|------|-------------|
| **VPSoft.Domain** | Modeles, DTOs, enums, interfaces | `Models/`, `DTOs/`, `Enums/` |
| **VPSoft.Services** | Logique metier (implementations) | Implementations des services |
| **VPSoft.Services.Abstractions** | Contrats de services (interfaces) | Interfaces avec marquage MEF |
| **VPSoft.Presentation** | Controllers API | `Controllers/` (26 controllers) |
| **VPSoft.Persistence** | NHibernate, repositories, migrations | `Mappings/`, `Repositories/`, `Migrations/` |
| **VPSoft.Web** | Host ASP.NET Core 9.0 + Vue.js 3 | `Program.cs`, DI config |

## Autres projets

| Projet | Role |
|--------|------|
| VPSoft.Compilation | Compilation dynamique du code C# injecte |
| VPSoft.Localization | Systeme de localisation/traduction |
| VPSoft.Mail | Envoi d'emails et templates |
| VPSoft.Reporting | Generation de rapports |
| VPSoft.Import | Import de donnees (CSV, Excel) |
| VPSoft.Export | Export de donnees |
| VPSoft.Scheduler | Planification des DynamicBatch |
| VPSoft.Security | Authentification et autorisation |
| VPSoft.FileStorage | Stockage de fichiers |
| VPSoft.Infrastructure | Utilitaires transverses |
| VPSoft.DI | Injection de dependances |

## Controllers principaux (VPSoft.Presentation)

| Controller | Endpoints | Description |
|-----------|-----------|-------------|
| `DynamicApiController` | api/v1/Post, Patch, Put, Delete | REST CRUD par entite |
| `VP2Controller` | Api/V2/VP/* | API V2 (GetManySelect, GetCount, etc.) |
| `AppReflectionController` | Api/V2/VP/AppReflection/* | Reflection sur les entites |
| `DynamicFunctionController` | Api/V2/VP/Functions/* | Invocation de DynamicFunction |
| `DynamicBatchController` | Api/V2/VP/Batch/* | Execution de DynamicBatch |
| `DynamicPageController` | - | Rendu des pages custom |
| `ImportController` | Api/V2/VP/Import/* | Import de donnees |
| `ExportController` | Api/V2/VP/Export/* | Export de donnees |
| `UrlController` | Api/V2/VP/Url/* | Resolution d'URLs |
| `AuthController` | - | Authentification |
| `UserController` | Api/V2/VP/User/* | Gestion utilisateurs |

## Modeles domaine cles (VPSoft.Domain)

### Entites de configuration (Builder)
- `DynamicSettings` — Configuration d'entite (EntityName, 12 champs code C#, 15 front)
- `DynamicField` — Definition de champ (69KB de code, tres complexe)
- `DynamicFieldName` — Labels localises par culture
- `DynamicFieldRole` — Permissions par role et champ
- `DynamicForm`, `DynamicSection` — Layout formulaire
- `DynamicButton` — Boutons dynamiques avec code
- `DynamicModule`, `DynamicFolder` — Arborescence de navigation
- `DynamicPage`, `DynamicPageField` — Pages custom
- `DynamicFunction`, `DynamicClass` — Code C# reutilisable
- `DynamicBatch` — Traitements planifies
- `DynamicConst` — Constantes
- `DynamicGlobalCode` — Code global app/module
- `DynamicWidget` — Widgets dashboard

### Entites de workflow
- `EntityWorkflow` — Definition de workflow
- `EntityWorkflowStatus` — Etats du workflow
- `EntityWorkflowAction` — Transitions
- `EntityWorkflowCondition` — Conditions sur les transitions
- `EntityWorkflowSet` — Affectations lors des transitions
- `EntityWorkflowSetByScript` — Affectations par script C#
- `EntityWorkflowFieldRole` — Permissions par etat

### Entites de conditionnalite
- `Conditionality` — Regle de conditionnalite
- `ConditionalityAction` — Action declenchee

### Entites d'audit
- `EntityAudit` — Classe de base (IsAuditData, IsAuditTrail)
- `AuditData` — Modifications champ par champ
- `AuditTrail` — Journal d'acces
- `AuditVersion` — Versions d'entites
- `AppLog` — Erreurs applicatives
- `MailLog` — Logs email
- `CascadeEntityProcessLog` — Operations cascade

## Patterns d'authentification

- `[Authorize]` — Necessite un Bearer JWT valide
- `[AllowAnonymous]` — Acces libre
- `[ApiResource]` — Attribut custom pour les ressources API
- Auth OAuth2 via Keycloak (client_credentials)

## Migrations (VPSoft.Persistence)

FluentMigrator, versions V10_0 a V10_3 :
- `Migrations/V10_0/` — Schema initial
- `Migrations/V10_1/` — Evolutions
- `Migrations/V10_2/` — Evolutions
- `Migrations/V10_3/` — Derniere version

## Commandes utiles pour explorer (si code source local disponible)

```bash
# Chercher un terme dans le code C#
grep -r "MonTerme" <CHEMIN_VPSOFT_DEV>/ --include="*.cs"

# Trouver un modele
grep -r "class MonModele" <CHEMIN_VPSOFT_DEV>/ --include="*.cs"

# Voir un controller
cat <CHEMIN_VPSOFT_DEV>/VPSoft.Presentation/Controllers/VP2Controller.cs

# Lister les migrations
ls <CHEMIN_VPSOFT_DEV>/VPSoft.Persistence/Migrations/
```

Remplacer `<CHEMIN_VPSOFT_DEV>` par le chemin local du code source si disponible.
