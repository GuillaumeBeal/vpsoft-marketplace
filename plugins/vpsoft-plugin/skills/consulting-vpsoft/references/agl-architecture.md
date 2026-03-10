# Architecture AGL VPSoft

Reference du modele AGL (Application Generation Layer). Toute la configuration est en base de donnees et accessible via `get_many_select`.

## Hierarchie AGL complete

```
APPLICATION
│
├── DynamicGlobalCode (DynamicModule=null)
│   └── CodeRazor, CodeCss, CodeJavascript              ← Code app-level
│
├── DynamicModule ─────────────────────────────────────── MODULE METIER
│   ├── Code, LocalizedName, Color, IsTransverse
│   ├── DynamicGlobalCode (DynamicModule=X)              ← Code module-level
│   │
│   └── DynamicFolder ────────────────────────────────── DOSSIER
│       ├── Code, LocalizedName, ParentFolder (FK recursive)
│       │
│       └── DynamicSettings ──────────────────────────── ENTITE/TABLE
│           ├── EntityName, LocalizedEntityName
│           ├── DynamicSettingsType: Standard|Custom|View
│           ├── EntityFunctionality: Module|Parameter
│           ├── 12 champs code C# (voir code-injection.md)
│           ├── 15 champs code front (voir code-injection.md)
│           │
│           ├── DynamicForm ──────────────────────────── FORMULAIRE
│           │   └── DynamicSection ────────────────────── SECTION
│           │       ├── Code, LocalizedName, OrderBy
│           │       └── DynamicField ──────────────────── CHAMP
│           │           ├── PropertyName, FormPropertyType
│           │           ├── Required, IsEntityCode, IsEntityName
│           │           ├── DynamicFieldName[] (labels)
│           │           └── DynamicFieldRole[] (permissions)
│           │
│           ├── DynamicButton[] ───────────────────────── BOUTONS
│           ├── Conditionality[] ──────────────────────── CONDITIONNALITES
│           ├── EntityWorkflow ────────────────────────── WORKFLOW
│           └── TreeConf ──────────────────────────────── ARBRE HIERARCHIQUE
│
├── DynamicFunction ───────────────────────────────────── FONCTIONS C#
├── DynamicClass ──────────────────────────────────────── CLASSES C#
├── DynamicBatch ──────────────────────────────────────── TRAITEMENTS PLANIFIES
├── DynamicConst ──────────────────────────────────────── CONSTANTES
├── DynamicPage ───────────────────────────────────────── PAGES CUSTOM
└── DynamicWidget ─────────────────────────────────────── WIDGETS DASHBOARD
```

## DynamicSettings — Entite/Table

Champs principaux :
- `EntityName` : Nom technique (utilise dans les requetes API)
- `LocalizedEntityName` : Nom metier affiche a l'utilisateur
- `ViewName` : null pour les vraies tables, non-null pour les vues
- `DynamicSettingsType` : Standard (entite classique), Custom (herite d'une autre), View (vue)
- `EntityFunctionality` : Module (entite metier) ou Parameter (parametre/referentiel)
- `Status` : FormPropertyStatus (New, Active, etc.)
- `TypeFullName` : Nom complet du type C# genere

Requete pour lister :
```
get_many_select("DynamicSettings",
  "EntityName, LocalizedEntityName, DynamicSettingsType, EntityFunctionality, DynamicFolder.DynamicModule.Code",
  where='EntityState == "Active" && ViewName == null', take=100)
```

## DynamicField — Champ

Champs principaux :
- `PropertyName` : Nom technique du champ
- `FormPropertyType` : Type du champ (enum)
- `Required` : Champ obligatoire
- `IsEntityCode` : Code unique de l'entite (utilise pour la resolution FK via _Code_)
- `IsEntityName` : Nom principal de l'entite
- `IsEntityOrder` : Champ de tri par defaut
- `Search, Report, ImportExport` : Booleans de visibilite
- `DynamicPageFieldAdd/Edit/Details` : Code Razor injecte sur le champ

Types de champs (FormPropertyType) :
| Type | Description |
|------|-------------|
| String | Texte libre |
| Integer | Entier |
| Decimal | Nombre decimal |
| DateTime | Date et heure |
| Date | Date seule |
| Boolean | Vrai/faux |
| Entity | Relation FK vers une autre entite |
| DynamicEnum | Liste de valeurs dynamique |
| DynamicMultiEnum | Liste multi-selection |
| Enum | Enumeration C# |
| MultiCulture | Texte localise |
| AutoIncrement | Compteur auto-incremente |
| Guid | Identifiant unique |
| EntityComment | Commentaires lies a l'entite |

⚠ **Limitations API sur DynamicField** :
- `get_entity_fields("DynamicField")` retourne un tableau `fields` vide (entite framework interne)
- `get_many_select` ne supporte qu'un sous-ensemble de proprietes dans le SELECT :
  - **Valides** : PropertyName, PropertyDisplayName, Id, EntityName, DynamicSettings.EntityName
  - **Invalides (400)** : OrderBy, OrderForm, DynamicSection, DynamicFieldName (collections/many-to-many)
- Pour obtenir OrderBy ou le mapping champ→section, consulter le code source C# si disponible

## DynamicSection — Sections de formulaire

Sections qui regroupent les champs dans le formulaire d'une entite.

Requete :
```
get_many_select("DynamicSection",
  "Code, LocalizedName, OrderBy",
  where='DynamicSettings.EntityName == "MonEntite" && EntityState == "Active"',
  order_by="OrderBy", take=20)
```

- `Code` : Identifiant technique (parfois un GUID, parfois un nom lisible comme "Questionary")
- `LocalizedName` : Nom affiche (ex: "Formulaire", "Audit")
- `OrderBy` : Ordre d'affichage dans le formulaire (1, 2, 3...)
- FK : `DynamicSettings` (via DynamicSettings.EntityName)

⚠ **Relation many-to-many DynamicSection ↔ DynamicField** :
- La table de jointure est `DynamicSectionsFields` (definie dans NHibernate `DynamicSectionMap.cs` via `HasManyToMany`)
- Cette table de jointure n'est PAS exposee comme entite dans l'API V2
- DynamicField n'a pas de FK directe vers DynamicSection
- Il est donc **impossible de requeter** le lien champ → section via l'API
- **Workaround** : Inferer le mapping par connaissance metier ou lire le code source C#:
  - `VPSoft.Domain/Models/Builder/DynamicSection.cs` : `ISet<DynamicField> DynamicFields` (ligne 78)
  - `VPSoft.Persistence/NHibernate/Mapping/Builder/DynamicSectionMap.cs` : `HasManyToMany(x => x.DynamicFields).Table("DynamicSectionsFields")` (lignes 32-37)

## DynamicFieldName — Labels localises

Un champ peut avoir plusieurs labels selon la culture :
```
get_many_select("DynamicFieldName",
  "Name, ColumnName, Tooltip, DynamicField.PropertyName",
  where='DynamicField.EntityName == "MonEntite" && CultureCode == "fr-FR" && EntityState == "Active"', take=200)
```

Methode alternative (recommandee) — via `get_entity_fields` :
```
get_entity_fields("MonEntite")
→ Chaque champ retourne un dict LocalizedNames avec cles par CultureCode :
  { "fr-FR": "Intitule", "en-US": "Title" }
→ Cet outil requete en interne DynamicField + DynamicFieldName automatiquement
```

- `Name` : Label affiche sur le formulaire
- `ColumnName` : Entete de colonne dans la liste
- `Tooltip` : Info-bulle

## DynamicFieldRole — Permissions par role

Chaque champ peut avoir des permissions differentes par role :
```
get_many_select("DynamicFieldRole",
  "DynamicField.PropertyName, RoleInModule.Name, CanRead, RoleActionForAdd, RoleActionForEdit, Table, EditInLine, Filter, Api",
  where='DynamicField.EntityName == "MonEntite" && EntityState == "Active"', take=200)
```

Valeurs numeriques retournees par l'API :
| Champ | Valeur | Signification |
|-------|--------|---------------|
| RoleActionForAdd/Edit | 0 | Inactive — champ non visible |
| RoleActionForAdd/Edit | 1 | Active — champ editable |
| RoleActionForAdd/Edit | 2 | ActiveAndHidden — champ present mais cache |
| RoleActionForAdd/Edit | 3 | ReadOnly — champ visible mais non modifiable |
| Table | 0 | Inactive — pas dans la liste |
| Table | 1 | Display — affiche dans la liste |
| Table | 2 | Displayable — disponible mais pas affiche par defaut |
| CanRead | true/false | Lecture autorisee |
| Filter | true/false | Disponible comme filtre |

## Conditionality — Conditionnalites

Regles dynamiques qui modifient le comportement du formulaire :
```
get_many_select("Conditionality", "Name, Code, Condition, Type",
  where='EntityName == "MonEntite" && EntityState == "Active"', take=20)
get_many_select("ConditionalityAction",
  "ActivationRule, ActionRule, Event, Type, ActionData",
  where='Conditionality.EntityName == "MonEntite" && EntityState == "Active"', take=50)
```

Types d'actions : Filter, Validation, Hide, Show, Affectation, ReadOnly, Mandatory
Types d'evenements : Entity, Alerte, Form, Field, AffectationRule, NavMenu, Section, DynamicButton

## EntityWorkflow — Workflow

Systeme de workflow avec etats, transitions, conditions et affectations :
```
# Etats
get_many_select("EntityWorkflowStatus", "Code, LocalizedName, EntityWorkflow.EntityName",
  where='EntityWorkflow.EntityName == "MonEntite" && EntityState == "Active"', take=20)

# Actions/Transitions
get_many_select("EntityWorkflowAction",
  "Code, LocalizedName, Action, EntityWorkflowStatus.Code, NextEntityWorkflowStatus.Code",
  where='EntityWorkflow.EntityName == "MonEntite" && EntityState == "Active"', take=20)

# Conditions sur une action
get_many_select("EntityWorkflowCondition",
  "PropertyName, Operator, ValueToCompare",
  where='EntityWorkflowAction.EntityWorkflow.EntityName == "MonEntite" && EntityState == "Active"', take=50)

# Permissions par etat (EntityWorkflowFieldRole)
# ⚠ Non accessible via API V2 — consulter l'interface VPSoft ou le code source C#.
```

## DynamicEnum — Listes de valeurs

⚠ DynamicEnum et DynamicEnumValue ne sont pas accessibles via l'API V2 (entites framework internes).
Pour decouvrir les valeurs d'un champ enum, echantillonner les donnees de l'entite :
```
get_entity_sample_data("MonEntite", "Code, MonChampEnum", limit=20)
get_many_select("MonEntite", "MonChampEnum",
  where='EntityState == "Active" && MonChampEnum != null', take=100)
```
