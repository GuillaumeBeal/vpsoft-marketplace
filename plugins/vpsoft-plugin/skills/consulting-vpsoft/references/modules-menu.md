# Modules, Menus et Pages dynamiques — VPSoft (Framework)

Reference du modele de navigation applicative VPSoft : modules, dossiers (DynamicFolder), pages dynamiques (DynamicPage) et controle d'acces par role.

⚠ Les modules, dossiers et pages varient d'une application a l'autre. Ce document decrit le **squelette framework** et comment l'explorer. Utiliser les requetes MCP ci-dessous pour decouvrir le contenu de l'instance connectee.

## Schema de donnees de la navigation

```
DynamicModule
│  Code, OrderBy, Color, LocalizedName
│  → Definit un module metier dans la barre laterale
│
└── DynamicFolder (arbre recursif via ParentFolder)
    │  Code, LocalizedName, ParentFolder (FK recursive), Id
    │  → Forme l'arborescence de menu a l'interieur d'un module
    │  → Chaque module a un dossier racine (souvent un GUID auto-genere)
    │  → Les sous-dossiers forment les niveaux du menu
    │
    ├── DynamicSettings (entites configurees)
    │   │  EntityName, LocalizedEntityName, DynamicFolder.Code
    │   │  → Les entites CRUD standard sont rangees dans des dossiers
    │   └── Chaque entite dans un dossier genere une entree menu (liste, create, edit)
    │
    └── DynamicPage (pages custom Razor/JS/CSS)
       │  Code, Name, DynamicFolder.Code, Description, Id
       │  CodeRazor, CodeJavascript, CodeCss (contenu code)
       │  → Pages custom rendues cote serveur (dashboards, widgets, cartes, outils)
       │
       └── VP.Functions.Invoke("FunctionName", ...) dans le code Razor/JS
           → Appelle des DynamicFunction C# depuis la page
```

### Champs valides par entite framework

Ces entites sont internes au framework (non listees par `get_all_entities()`, `get_entity_fields()` retourne vide).
Les champs sont decouverts par probing sur `get_many_select`.

| Entite | Champs valides | Champs invalides (400) |
|--------|----------------|----------------------|
| DynamicModule | Code, OrderBy, Color, LocalizedName, LocalizedDescription, Id, DynamicFolders, CultureParameters | Name, Description, DefaultPage, Icon, IsActive, Roles |
| DynamicFolder | Code, LocalizedName, ParentFolder, DynamicModule, DynamicModule.Code, Id, ViewName, CultureParameters | Name, ParentCode, OrderBy, Icon, IsVisible, IsActive, Roles, DynamicPages, DynamicPage, Permission |
| DynamicPage | Code, Name, DynamicFolder.Code, Description, ViewName, Id, CreatedDate, ModifiedDate, DeletedDate, CodeRazor, CodeJavascript, CodeCss, LocalizedName | Roles, IsVisible, Permission, DynamicModule, Module |
| AppRole | Name, Id | Code (n'existe pas) |
| DynamicSection | Code, LocalizedName, OrderBy, DynamicSettings.EntityName, Id | DynamicFields (collection many-to-many, non requetable) |
| DynamicField | PropertyName, PropertyDisplayName, Id, EntityName, DynamicSettings.EntityName | OrderBy, OrderForm, DynamicSection, DynamicFieldName (400 Bad Request) |
| DynamicFieldRole | DynamicField.PropertyName, DynamicField.EntityName, RoleInModule.Name, RoleInModule.Code, CanRead, RoleActionForAdd, RoleActionForEdit, Table, EditInLine, Filter, Api | (tous les champs utiles sont accessibles) |
| DynamicFieldName | Name, ColumnName, Tooltip, DynamicField.PropertyName, DynamicField.EntityName, CultureCode | (tous les champs utiles sont accessibles) |
| RoleInModule | Name, Code, DynamicModule.Code | (tous les champs utiles sont accessibles) |

### Notes sur les entites framework

- **DynamicFolder.ParentFolder** : retourne l'objet parent complet (pas juste un code). C'est la seule facon d'obtenir la hierarchie.
- **DynamicFolder.Code** : parfois un nom lisible ("System", "BDD"), parfois un GUID auto-genere pour les dossiers racine de module.
- **DynamicPage.DeletedDate** : `null` = page active. Filtrer `DeletedDate == null` au lieu de `EntityState == "Active"` (pas de champ EntityState sur DynamicPage).
- **DynamicModule.DynamicFolders** : retourne la liste complete des dossiers enfants.
- **DynamicPage.DynamicFolder.Code** : lien vers le dossier parent. Pas de FK inverse (on ne peut pas lister les pages depuis un dossier directement, il faut filtrer cote page).

---

## Controle d'acces

⚠ VPSoft a **trois niveaux de roles** qu'il ne faut pas confondre :

### 1. Roles applicatifs globaux (AppRole)

Roles definis au niveau de l'application entiere. Chaque utilisateur a un ou plusieurs AppRole.

```
# Decouvrir les roles globaux de l'application
get_many_select("AppRole", "Name, Id", take=10)

# Utilisateurs et leurs roles globaux
get_many_select("User", "UserName, Email, FullName, AppRoles, IsAdmin",
  where='EntityState == "Active"', take=50)
```

Les AppRole sont utilises pour l'acces global (connexion, modules visibles). Ils sont geres par Keycloak (realm roles, client roles).

### 2. Roles par module (RoleInModule)

Chaque DynamicModule definit ses propres roles via l'entite `RoleInModule`. Ces roles sont **rattaches a un module** (FK vers DynamicModule) et sont independants des AppRole.

```
# Decouvrir tous les RoleInModule avec leur module
get_many_select("RoleInModule", "Name, Code, DynamicModule.Code", take=50)

# Roles d'un module specifique
get_many_select("RoleInModule", "Name, Code",
  where='DynamicModule.Code == "MonModule"', take=20)
```

### 3. Permissions par champ d'entite (DynamicFieldRole)

Le troisieme niveau utilise `DynamicFieldRole` pour mapper un RoleInModule a des permissions sur les **champs** d'une entite. C'est le controle le plus fin : par champ, par role, avec lecture/ajout/edition.

```
# Permissions d'un RoleInModule sur les champs d'une entite
get_many_select("DynamicFieldRole",
  "DynamicField.PropertyName, RoleInModule.Name, CanRead, RoleActionForAdd, RoleActionForEdit, Table, Filter",
  where='DynamicField.EntityName == "MonEntite" && EntityState == "Active"', take=200)

# Lister les roles distincts utilises sur une entite
# → Extraire les valeurs uniques de RoleInModule.Name dans le resultat
```

| Champ | Valeurs |
|-------|---------|
| RoleActionForAdd/Edit | Inactive, Active, ActiveAndHidden, ReadOnly |
| Table | Inactive, Display, Displayable |
| CanRead | true/false |
| Filter | true/false |

**Hierarchie complete :** AppRole (global) → RoleInModule (par module) → DynamicFieldRole (par champ d'entite)
Cela permet de repondre a : "quels champs un role X du module Y voit-il en creation/edition/liste pour l'entite Z ?"

### Mapping role → menu

**Le mapping role → dossier/page n'est PAS expose via l'API V2.**
L'acces aux modules et dossiers est gere par :
- La configuration Keycloak (realm roles, client roles)
- Le code applicatif (souvent dans le code Razor des pages)

⚠ L'entite `Conditionality` a un champ `Type` qui utilise des **valeurs numeriques** (0, 1, etc.), pas des strings. Il n'y a pas de type "NavMenu" expose directement. Pour identifier les conditionnalites liees a la navigation :
```
get_many_select("Conditionality", "Name, Code, Condition, Type, EntityName",
  where='EntityState == "Active"', take=50)
# → Examiner les resultats pour trouver des conditions liees a la navigation
```

### 3. Permissions workflow par role (FormWorkflowByRole)

Pour le module Form, le workflow peut definir des permissions par role et par etape :

```
get_many_select("FormWorkflowByRole",
  "FormWorkflow.Code, Role.Name, Validate, Reject, CanGoFirst, CanGoLast, CanVersion",
  where='FormWorkflow.Form.Code == "MON_FORM" && EntityState == "Active"', take=50)
```

---

## Conventions de nommage des pages

Les pages dynamiques suivent des conventions de nommage qui permettent de distinguer les points d'entree utilisateur des composants techniques :

### Points d'entree (pages visibles dans le menu)

| Pattern | Type | Description |
|---------|------|-------------|
| `Root*`, `Dashboard*`, `Home*` | Page d'accueil du module | Page principale visible a l'ouverture du module |
| `LayoutHome*`, `LayoutLandingPage`, `LayoutDashboard` | Dashboard/accueil | Variantes de pages d'accueil avec prefix Layout |
| `LayoutESRS`, `Layout[Domaine]` | Page metier principale | Points d'entree par domaine fonctionnel |
| `FIB` (sans suffixe) | Fiche principale | Page de detail multi-onglets (Fiche Immeuble Base) |
| `*Map*` | Cartographie | Pages avec carte Leaflet/Google Maps |
| `Kanban*`, `*Vue` (avec KanBan) | Vue Kanban | Tableaux kanban interactifs |
| `*List`, `*Manager` | Liste/gestion | Pages de gestion avec grille de donnees |
| `*Table` (sans Debug) | Vue tabulaire | Pages de donnees en format tableau |

### Composants (inclus dans d'autres pages, pas des points d'entree)

| Pattern | Type | Description |
|---------|------|-------------|
| `Modal*` | Modale | Fenetre modale (edition, detail, confirmation) |
| `*Widget*` | Widget | Composant visuel inclus dans un dashboard |
| `Partial*`, `Layout*Card*`, `Layout*Breadcrumb*` | Vue partielle | Fragment de page inclus via @Html.Partial |
| `*Helper*` | Helper JS/Razor | Code utilitaire inclus par d'autres pages |
| `*Filter*` | Filtre | Composant de filtrage reutilisable |
| `*Toolbar*`, `*SideBar*` | Barre d'outils | Composant de navigation/actions |
| `*Indicator*`, `*Options` | Indicateur/options | Composant d'affichage ou de parametrage |
| `FIB*` (avec suffixe) | Onglet FIB | Sous-pages/onglets de la fiche immeuble |
| `Layout[Entite]` (sans Home/Dashboard) | Layout de formulaire | Mise en page d'une entite (LayoutForm, LayoutActions) |

### Pages metier specifiques (varient par application)

| Pattern | Type | Description |
|---------|------|-------------|
| `Export*` | Export | Pages d'export de donnees (Excel, PDF, rapports) |
| `AI*`, `Sandbox*AI` | Intelligence artificielle | Pages IA (chat, generation, debug IA) |
| `Generate*` | Generation | Pages de generation automatique de contenu |
| `BanqueExigence_*` | Module metier | Pages liees a un domaine metier specifique (ex: banque d'exigences) |
| `EntityForm*` | Formulaire entite | Pages de formulaire personnalise pour une entite |

### Pages techniques (developpement/administration)

| Pattern | Type | Description |
|---------|------|-------------|
| `Turbo*` | Outil developpeur | IDE, traducteur, form builder |
| `Playground*`, `test*`, `Debug*` | Developpement/debug | Pages de test, bac a sable |
| `Sandbox*` (sans AI) | Bac a sable | Pages de prototypage |
| `Mapping_*`, `Remapping*` | Administration | Pages de mapping systeme |
| `*Batch*`, `batch*` | Traitement par lots | Execution de batchs de donnees |
| `Migration*`, `Refresh*` | Migration | Scripts de migration de donnees |
| `Script*` | Script ponctuel | Scripts ad-hoc (avec souvent une date dans le nom) |
| `Duplicate*`, `Delete*`, `Deploy*` | Operations en masse | Actions bulk sur les donnees |
| `Spec*` | Specifications | Outils de specification des entites |
| `DTO*`, `Enum*` | Utilitaires dev | Generateurs de code technique |

---

## Relations inter-pages

### VP.Functions.Invoke

Les pages appellent des DynamicFunction C# via `VP.Functions.Invoke("FunctionName", ...)` dans leur code Razor ou JavaScript. C'est le mecanisme principal de liaison page→logique metier.

Pour analyser les relations :

```
# 1. Recuperer le code Razor/JS d'une page
get_many_select("DynamicPage", "Code, CodeRazor, CodeJavascript",
  where='Code == "MaPage"')

# 2. Chercher VP.Functions.Invoke dans le code retourne
#    Pattern regex : VP\.Functions\.Invoke\s*\(\s*"([^"]+)"

# 3. Lister les DynamicFunction disponibles
get_many_select("DynamicFunction", "Name, Code, Description, IsAsync",
  where='EntityState == "Active"', take=200)
```

### Navigation inter-pages

Les pages peuvent naviguer entre elles via le pattern `DynamicPage/{PageCode}` dans le code Razor/JS :

```
# Chercher dans le code : DynamicPage/([A-Za-z0-9_]+)
# Exemples typiques :
#   href="/DynamicPage/Edit"
#   window.location = "/DynamicPage/AdvancedSearchResults"
```

---

## Requetes pour explorer la navigation

### Decouverte des modules

```
# Tous les modules avec leur ordre d'affichage
get_many_select("DynamicModule", "Code, OrderBy, Color, LocalizedName", take=50)
```

### Decouverte des dossiers (menu)

```
# Tous les dossiers avec leur module
get_many_select("DynamicFolder", "Code, LocalizedName, DynamicModule.Code", take=200)

# Hierarchie parent-enfant des dossiers (reconstruit l'arbre du menu)
get_many_select("DynamicFolder", "Code, ParentFolder, DynamicModule.Code", take=200)
# Note : ParentFolder retourne l'objet complet du parent

# Dossiers d'un module specifique
get_many_select("DynamicFolder", "Code, LocalizedName, ParentFolder",
  where='DynamicModule.Code == "MonModule"', take=100)
```

### Decouverte des pages

```
# Toutes les pages actives avec leur dossier
get_many_select("DynamicPage", "Code, Name, DynamicFolder.Code",
  where='DeletedDate == null', take=200)

# Pages d'un dossier specifique
get_many_select("DynamicPage", "Code, Name",
  where='DynamicFolder.Code == "MonDossier" && DeletedDate == null', take=50)

# Pages d'un module (via le dossier)
get_many_select("DynamicPage", "Code, Name, DynamicFolder.Code",
  where='DynamicFolder.DynamicModule.Code == "MonModule" && DeletedDate == null', take=100)

# Contenu code d'une page
get_many_select("DynamicPage", "Code, CodeRazor, CodeJavascript, CodeCss",
  where='Code == "MaPage"')
# ⚠ CodeRazor peut etre tres volumineux — batacher avec skip/take si necessaire
```

### Decouverte des entites dans les menus

```
# Entites rangees dans un dossier (genere des entrees menu CRUD)
get_many_select("DynamicSettings",
  "EntityName, LocalizedEntityName, DynamicFolder.Code, DynamicFolder.DynamicModule.Code",
  where='EntityState == "Active" && ViewName == null', take=200)

# Entites d'un module specifique
get_many_select("DynamicSettings",
  "EntityName, LocalizedEntityName, DynamicFolder.Code",
  where='EntityState == "Active" && DynamicFolder.DynamicModule.Code == "MonModule"', take=100)
```

### Analyse des roles et permissions (3 niveaux)

```
# Niveau 1 — Roles globaux (AppRole)
get_many_select("AppRole", "Name, Id", take=10)

# Utilisateurs et leurs roles globaux
get_many_select("User", "UserName, FullName, AppRoles, IsAdmin",
  where='EntityState == "Active"', take=50)

# Niveau 2 — Roles par module (RoleInModule)
get_many_select("RoleInModule", "Name, Code, DynamicModule.Code", take=50)

# Niveau 3 — Permissions par champ (DynamicFieldRole)
get_many_select("DynamicFieldRole",
  "DynamicField.PropertyName, RoleInModule.Name, CanRead, RoleActionForAdd, RoleActionForEdit, Table, Filter",
  where='DynamicField.EntityName == "MonEntite" && EntityState == "Active"', take=200)
```

---

## Workflow de cartographie complete d'une application

Pour produire un diagramme complet des pages dynamiques et de leurs relations :

```
1. Lister les modules :
   get_many_select("DynamicModule", "Code, OrderBy, LocalizedName", take=50)

2. Lister les dossiers avec hierarchie :
   get_many_select("DynamicFolder", "Code, LocalizedName, ParentFolder, DynamicModule.Code", take=200)

3. Lister les pages actives :
   get_many_select("DynamicPage", "Code, Name, DynamicFolder.Code",
     where='DeletedDate == null', take=200)

4. Pour chaque page, recuperer le code (batches de 30 pour eviter les timeouts) :
   get_many_select("DynamicPage", "Code, CodeRazor",
     where='DeletedDate == null', skip=0, take=30)
   get_many_select("DynamicPage", "Code, CodeRazor",
     where='DeletedDate == null', skip=30, take=30)
   ...

5. Extraire les VP.Functions.Invoke du code Razor/JS :
   Pattern : VP\.Functions\.Invoke\s*\(\s*"([^"]+)"

6. Extraire les navigations inter-pages :
   Pattern : DynamicPage/([A-Za-z0-9_]+)

7. Lister les DynamicFunction :
   get_many_select("DynamicFunction", "Name, Code, Description",
     where='EntityState == "Active"', take=200)

8. Generer un diagramme Mermaid :
   - Sous-graphes par module
   - Pages comme noeuds
   - Fonctions invoquees comme liens
   - Navigation inter-pages comme fleches
```

⚠ **Session MCP** : Le serveur MCP peut necessiter une session fraiche pour chaque requete independante.
Si les resultats reviennent vides apres une premiere requete reussie, creer une nouvelle session MCP.
