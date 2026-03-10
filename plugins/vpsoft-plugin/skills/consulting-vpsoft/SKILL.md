---
name: consulting-vpsoft
description: "Exploration et comprehension d'une application VPSoft via le serveur MCP. Se declenche des que l'utilisateur mentionne : VPSoft, entite, DynamicSettings, DynamicField, champ, table, donnees metier, code dynamique, DynamicFunction, DynamicBatch, DynamicPage, DynamicConst, DynamicClass, injection code, erreur applicative, audit, log, convention _Code_, EntityState, AGL, workflow, conditionnalite, module, dossier, formulaire, section, bouton dynamique, role, permission, diagramme, schema, mermaid, form, questionnaire, campagne, FormQuestion, FormFormula, FormAnswer, Section, Campaign, Questionary, reponse, formule, deploiement, import, export, fichier import, xlsx, ESRS, CSRD, DataPoint, culture, langue, multi-langue, Culture, validation import, erreur import, documentation formulaire, comparaison version, diff version, orchestration, export formulaire."
version: 1.0.0
---

# Consulting VPSoft — Exploration et comprehension d'une application

Skill pour explorer une application VPSoft via le serveur MCP. VPSoft est une plateforme AGL (Application Generation Layer) ou toute la configuration — entites, champs, code metier, workflows, conditionnalites, permissions — est stockee en base de donnees et accessible via les outils MCP.

**Fichiers de reference a consulter selon le besoin :**
- `references/tools-catalog.md` — Catalogue des 15 outils MCP avec parametres et exemples
- `references/where-syntax.md` — Syntaxe WHERE System.Linq.Dynamic.Core, pieges et exemples
- `references/agl-architecture.md` — Modele complet AGL : DynamicSettings, DynamicField, modules, workflows
- `references/code-injection.md` — 38+ points d'injection code (12 C#, 15 CSS/JS/Razor, champs, boutons)
- `references/csharp-source.md` — Cartographie du code source C# (19 projets, controllers, services)
- `references/form-module.md` — Module Form : questionnaires, campagnes, formules, import/export, validation, erreurs, 40+ entites
- `references/modules-menu.md` — Navigation applicative : modules, dossiers, pages, trois niveaux de roles (AppRole → RoleInModule par module → DynamicFieldRole par champ), conventions de nommage, requetes de decouverte

---

## Regles fondamentales

1. **EntityState obligatoire** : Toujours filtrer `EntityState == "Active"` dans les requetes WHERE pour exclure les donnees archivees/supprimees. Valeurs possibles : 0=Draft, 1=Active, 2=Archived, 3=Deleted.
2. **Egalite stricte** : Utiliser `==` et jamais `=` dans les clauses WHERE.
3. **Strings entre double quotes** : `Name == "Test"` (jamais de guillemets simples).
4. **Verifier les champs** : Toujours appeler `get_entity_fields()` avant de construire une requete pour connaitre les PropertyName exacts.
5. **FK en lecture** : Notation point dans SELECT — `Organization.Code, Organization.Name`.
6. **FK en ecriture** : Suffixe `_Code_` dans le body POST — `Organization_Code_: "BOULOGNE"`. La valeur correspond au champ avec `IsEntityCode=true` sur l'entite liee.
7. **Labels metier** : Les EntityName/PropertyName sont techniques. Toujours recuperer les noms localises (LocalizedEntityName, DynamicFieldName) pour comprendre le sens metier.
8. **CRUD limite** : Seules 3 entites ont le REST v1 active : Organization, Ticket, HSEIncident. Les 233 entites sont lisibles via l'API V2 (get_many_select).
9. **PATCH/PUT non fonctionnels** : Les endpoints Patch/Put retournent une erreur 400 sur toutes les entites. Utiliser uniquement Create et Delete.
10. **Ne pas creer de config** : Ne jamais utiliser `create_entity()` pour des entites de configuration (DynamicSettings, DynamicField, etc.) — le service-layer cascade n'est pas declenche par l'API v1.
11. **Entites specifiques a l'application** : Les entites metier (Building, Ticket, Space, MaintenanceEquipment, HSEIncident, etc.) varient d'une application a l'autre. VPSoft est un AGL : chaque application deploie ses propres entites. Exemple : ESG_V104 est une application ESRS/CSRD (DataPoint, Form, Campaign), pas de maintenance (Building, Ticket). Toujours commencer par `get_all_entities()` pour decouvrir les entites disponibles sur l'instance connectee.
12. **Langues actives (Culture)** : Chaque application a N langues actives. Les champs MultiCulture (Name, Description, Question, etc.) generent une colonne par langue dans les exports/imports. Decouvrir les langues actives : `get_many_select("Culture", "CultureCode, CultureCodeShort, IsDefault", where='EntityState == "Active"', take=20)`. L'entite `Culture` contient `CultureCode` (ex: "fr-FR"), `IsDefault` (langue par defaut), `IsLocal` (culture de base).

---

## Workflow 1 — Decouverte de l'application

Objectif : Comprendre les modules, dossiers, entites et pages de l'application.

**Consulter `references/modules-menu.md`** pour le modele de navigation (modules, dossiers, pages), les trois niveaux de roles (AppRole → RoleInModule → DynamicFieldRole), et les conventions de nommage pour identifier les points d'entree.

```
1. Lister toutes les entites :
   get_all_entities()
   → Retourne 233+ entites avec EntityName et LocalizedEntityName

2. Explorer l'arborescence modulaire :
   get_many_select("DynamicModule", "Code, OrderBy, LocalizedName, Color", take=50)
   get_many_select("DynamicFolder", "Code, LocalizedName, DynamicModule.Code", take=200)
   get_many_select("DynamicSettings",
     "EntityName, LocalizedEntityName, DynamicFolder.Code, DynamicFolder.DynamicModule.Code",
     where='EntityState == "Active" && ViewName == null', take=100)

3. Explorer les pages dynamiques visibles dans les menus :
   get_many_select("DynamicPage", "Code, Name, DynamicFolder.Code",
     where='DeletedDate == null', take=200)
   # Identifier les points d'entree : Root*, Dashboard*, Home*, FIB, *Map*, Kanban*
   # Distinguer des composants : *Widget*, Partial*, *Helper*, *Filter*

4. Verifier les roles et permissions (3 niveaux) :
   # Niveau 1 — Roles globaux (AppRole)
   get_many_select("AppRole", "Name, Id", take=10)
   # Niveau 2 — Roles par module (RoleInModule, rattaches a DynamicModule)
   get_many_select("RoleInModule", "Name, Code, DynamicModule.Code", take=50)
   # Niveau 3 — Permissions par champ d'entite (DynamicFieldRole)
   get_many_select("DynamicFieldRole",
     "DynamicField.PropertyName, RoleInModule.Name, CanRead, RoleActionForAdd, RoleActionForEdit",
     where='DynamicField.EntityName == "MonEntite" && EntityState == "Active"', take=200)
   ⚠ Le mapping role → dossier/page n'est pas expose via API (gere par Keycloak)

5. Pour chaque entite d'interet :
   get_entity_fields("MonEntite")          → champs, types, FK, required
   get_entity_sample_data("MonEntite", "Code, Name, FK.Name", limit=5)
```

---

## Workflow 2 — Exploration du modele de donnees

Objectif : Comprendre la structure detaillee d'une entite et ses relations.

```
1. Structure des champs :
   get_entity_fields("MonEntite")
   → Identifier :
     - IsEntityCode=true : code unique (utilise pour _Code_ en ecriture)
     - Required=true : champs obligatoires
     - PropertyType contenant "VPSoft" ou "Entity" : relations FK
     - PropertyType "DynamicEnum" : liste de valeurs
     - PropertyType "MultiCulture" : champ localise

2. Labels localises :
   get_many_select("DynamicFieldName",
     "Name, ColumnName, Tooltip",
     where='DynamicField.EntityName == "MonEntite" && EntityState == "Active"', take=100)

3. Visualiser les relations :
   get_entity_sample_data("MonEntite",
     "Code, Name, ParentEntity.Name, LinkedEntity.Code", limit=10)

4. Compter les enregistrements :
   get_entities_count("MonEntite", where='EntityState == "Active"')

5. Enumeration dynamique :
   # DynamicEnum/DynamicEnumValue ne sont pas accessibles via l'API V2.
   # Pour decouvrir les valeurs d'un champ enum, utiliser un echantillon :
   get_entity_sample_data("MonEntite", "Code, MonChampEnum", limit=20)
   # Ou lister les valeurs distinctes :
   get_many_select("MonEntite", "MonChampEnum",
     where='EntityState == "Active" && MonChampEnum != null', take=100)
```

---

## Workflow 3 — Lecture du code dynamique

Objectif : Lire et comprendre le code C#/JS/CSS injecte dans la configuration.

```
Code C# injecte sur une entite (12 points) :
  get_many_select("DynamicSettings",
    "EntityName, UsingsExpression, InitializeCreateExpression, InitializeEditExpression,
     InitializeDetailsExpression, ValidationCreateExpression, ValidationEditExpression,
     InjectionCreateExpression, InjectionEditExpression,
     PostCreateExpression, PostEditExpression,
     ListenerCreateExpression, ListenerEditExpression",
    where='EntityName == "MonEntite"')

Code JS/CSS/Razor sur une entite (15 points) :
  get_many_select("DynamicSettings",
    "EntityName, ContentJavascriptCreateExpression, ContentJavascriptEditExpression,
     ContentJavascriptDetailsExpression, ContentJavascriptListExpression, ContentJavascriptGlobalExpression,
     ContentCssCreateExpression, ContentCssEditExpression, ContentCssDetailsExpression,
     ContentCssListExpression, ContentCssGlobalExpression,
     ContentRazorCreateExpression, ContentRazorEditExpression, ContentRazorDetailsExpression,
     ContentRazorListExpression, ContentRazorGlobalExpression",
    where='EntityName == "MonEntite"')

Fonctions reutilisables (DynamicFunction) :
  get_many_select("DynamicFunction",
    "Name, Code, Description, IsAsync",
    where='EntityState == "Active"', take=50)
  # Puis lire le code d'une fonction :
  get_many_select("DynamicFunction",
    "Name, CodeUsing, CodeFunction, CodeClass",
    where='Name == "MaFonction"')

Traitements planifies (DynamicBatch) :
  get_many_select("DynamicBatch",
    "Name, Code, Description, ExecTime, CodeBatch, ClassBatch",
    where='EntityState == "Active"', take=50)

Classes partagees (DynamicClass) :
  get_many_select("DynamicClass",
    "Name, Code, CodeUsing, CodeClass",
    where='EntityState == "Active"', take=50)

Constantes (DynamicConst) :
  get_many_select("DynamicConst",
    "Name, Code, ConstType, CodeCSharp, Value",
    where='EntityState == "Active"', take=50)

Pages custom (DynamicPage) :
  get_many_select("DynamicPage",
    "Code, LocalizedName, CodeRazor, CodeJavascript, CodeCss",
    where='EntityState == "Active"', take=50)

Boutons dynamiques :
  get_many_select("DynamicButton",
    "Code, LocalizedName, DynamicExpression, JSCallBack, DynamicButtonPosition",
    where='DynamicSettings.EntityName == "MonEntite" && EntityState == "Active"', take=20)
```

Voir `references/code-injection.md` pour la liste complete des 38+ points d'injection.

---

## Workflow 4 — Requetage de donnees

Objectif : Construire des requetes correctes avec get_many_select.

```
Syntaxe WHERE (System.Linq.Dynamic.Core) :
  Egalite :       Name == "Test"
  Numerique :     Surface >= 100
  Booleen :       IsActive == true
  Null :          Description != null
  FK :            Organization.Name == "MonOrg"
  Date :          CreatedDate >= "2024-01-01T00:00:00"
  GUID :          Id == "550e8400-..."
  Combine :       EntityState == "Active" && Surface >= 100 && Organization.Name == "MonOrg"
  OU :            (Type == "Bureau" || Type == "Salle")
  NOT :           !(Type == "Archive")

Tri et pagination :
  get_many_select("Building", "Name, Surface, Organization.Name",
    where='EntityState == "Active" && Surface >= 100',
    order_by="Surface DESC", skip=0, take=20)
```

Voir `references/where-syntax.md` pour la reference complete avec pieges et erreurs courantes.

---

## Workflow 5 — Diagnostic et audit

Objectif : Investiguer les erreurs, modifications et acces.

```
Erreurs applicatives :
  get_app_errors(take=10)
  get_app_errors(search="NullReferenceException", date_from="2024-06-15T00:00:00")
  get_app_errors(error_type="0")    → erreurs gerees (try/catch)
  get_app_errors(error_type="1")    → erreurs non gerees (crashes)

Historique des modifications :
  get_audit_history(entity_name="Building", entity_id="GUID-ICI")
  get_audit_history(entity_name="DynamicSettings", field_name="ValidationCreateExpression")
  get_audit_history(action="1")     → uniquement les Updates

Journal d'acces utilisateur :
  get_audit_trail(user_name="Dupont", take=50)

Emails envoyes :
  get_mail_logs(to_address="user@example.com", date_from="2024-06-01T00:00:00")

Versions d'entite :
  get_audit_versions(entity_name="Building", entity_id="GUID-ICI")

Operations cascade :
  get_cascade_logs(status="5")      → status 5 = Error
  get_cascade_logs(operation_type="0")  → suppressions

Note : Les logs d'execution du code dynamique (DynamicPage.log, DynamicFunction.log,
DynamicBatch.log) sont sur le filesystem serveur, pas accessibles via MCP.
```

---

## Workflow 6 — Analyse du code source C#

Objectif : Comprendre le socle technique pour des investigations approfondies.

```
Projets principaux :
  VPSoft.Domain         → Modeles, DTOs, enums, interfaces
  VPSoft.Services       → Logique metier (implementations)
  VPSoft.Services.Abstractions → Contrats de services (interfaces)
  VPSoft.Presentation   → Controllers API (26 controllers, 500+ endpoints)
  VPSoft.Persistence    → NHibernate mappings, repositories, migrations
  VPSoft.Web            → Host ASP.NET Core 9.0 + Vue.js 3

Autres projets : Compilation, Localization, Mail, Reporting, Import, Export,
  Scheduler, Security, FileStorage, Infrastructure, DI.

Pour explorer le code source si un chemin local est disponible, utiliser Grep et Read.
```

Voir `references/csharp-source.md` pour la cartographie detaillee des projets, controllers et modeles domaine.

---

## Workflow 7 — Diagrammes Mermaid

Objectif : Generer des diagrammes visuels a partir des donnees VPSoft.

**Diagramme ER (relations entre entites)** :
```
1. Identifier les entites d'interet via get_all_entities()
2. Pour chaque entite, appeler get_entity_fields() et identifier les FK (PropertyType contenant "Entity")
3. Generer un diagramme Mermaid erDiagram avec les relations :

erDiagram
    Building ||--o{ Space : contient
    Building }o--|| Organization : appartient
    Space }o--|| SpaceType : "type"
```

**Diagramme Workflow (etats et transitions)** :
```
1. Recuperer les etats :
   get_many_select("EntityWorkflowStatus", "Code, LocalizedName, EntityWorkflow.EntityName",
     where='EntityWorkflow.EntityName == "MonEntite" && EntityState == "Active"', take=20)
2. Recuperer les transitions :
   get_many_select("EntityWorkflowAction",
     "Code, LocalizedName, EntityWorkflowStatus.Code, NextEntityWorkflowStatus.Code",
     where='EntityWorkflow.EntityName == "MonEntite" && EntityState == "Active"', take=20)
3. Generer un diagramme Mermaid stateDiagram-v2 :

stateDiagram-v2
    [*] --> Draft
    Draft --> Submitted : Soumettre
    Submitted --> Approved : Approuver
    Submitted --> Rejected : Rejeter
    Rejected --> Draft : Corriger
```

**Arborescence modulaire** :
```
1. Recuperer modules, dossiers et entites (voir Workflow 1)
2. Generer un diagramme Mermaid graph TD :

graph TD
    M[Module HSE] --> F1[Incidents]
    M --> F2[Equipements]
    F1 --> E1[HSEIncident]
    F1 --> E2[HSEAction]
    F2 --> E3[MaintenanceEquipment]
```

**Diagramme Form (structure questionnaire)** :
```
1. Recuperer le formulaire et ses sections (voir Workflow 8)
2. Generer un diagramme hierarchique des sections et questions
```

---

## Workflow 8 — Module Form (Questionnaires et Campagnes)

Objectif : Explorer la configuration des formulaires, questionnaires et campagnes.

Le module Form est un sous-systeme complet avec 40+ entites dediees, totalement separe du DynamicSettings.
**Consulter `references/form-module.md`** pour l'architecture complete, les champs de chaque entite, et 12+ exemples de requetes.

Entites principales : Form → Section → FormQuestion → FormAnswer, FormFormula, QuestionDisplayCondition.
Deploiement : Campaign → Questionary → UserResponse.
Listes preedefinies : ResponseListCategory → ResponseList.
Workflow : FormWorkflow, FormWorkflowByRole, FormWorkflowActionBySection.

```
# Point d'entree — lister les formulaires :
get_many_select("Form", "Code, Name, Version, Prefix, Description",
  where='EntityState == "Active"', take=50)

# Puis explorer sections, questions, reponses — voir references/form-module.md
```

---

## Workflow 9 — Import/Export de formulaires

Objectif : Transformer un fichier client chaotique en 3 fichiers Excel (.xlsx) importables dans VPSoft.

Le module Form dispose d'un import par fichier Excel pour 3 entites : **Section**, **FormQuestion**, **FormAnswer**. L'import ne fonctionne que sur les formulaires templates (non versionnes). Les codes doivent etre uniques dans le fichier ET en base. L'ordre d'import est : Sections → Questions → Reponses.

**Consulter `references/form-module.md` section Import/Export** pour :
- Colonnes exactes de chaque fichier et regles de validation
- **Validation pre-import** : checklist complete + requetes de verification
- **Guide des erreurs d'import** : tableau erreur → cause → solution
- **Workflow Export → Transform → Re-import** : cycle complet
- **Syntaxe des formules** : expressions C# (Roslyn), ConditionalDisplay, FormulaInterval
- Guide de transformation pas-a-pas

```
# Methodologie de transformation :
0. Decouvrir les langues actives (determine le nombre de colonnes multi-langue) :
   get_many_select("Culture", "CultureCode, CultureCodeShort, IsDefault",
     where='EntityState == "Active"', take=20)

1. Identifier le formulaire cible :
   get_many_select("Form", "Code, Name, Version, Prefix",
     where='EntityState == "Active"', take=50)

2. Verifier que c'est un template (Version 0 ou TemplateId vide)

3. Lister les types de questions disponibles :
   get_many_select("FormQuestionType", "Code, Name",
     where='EntityState == "Active"', take=50)

4. Verifier les codes existants (eviter les collisions) :
   get_many_select("Section", "Code", where='EntityState == "Active"', take=500)
   get_many_select("FormQuestion", "Code", where='EntityState == "Active"', take=500)

5. Lister les categories de reponses preedefinies :
   get_many_select("ResponseListCategory", "Code, Name",
     where='EntityState == "Active"', take=50)

6. Produire les 3 fichiers Excel :
   - Colonnes obligatoires en en-tete orange (#FF8C00) + gras
   - Colonnes optionnelles en en-tete gris clair (#D9D9D9)
   - Une colonne par langue active pour chaque champ multi-langue
   (voir references/form-module.md pour le schema complet)
```

---

## Workflow 10 — Generation de documentation formulaire

Objectif : Produire un document lisible (Word/PDF) decrivant la structure complete d'un formulaire, pour la recette client ou la documentation projet.

```
1. Extraire la structure complete du formulaire :
   get_many_select("Form", "Code, Name, Version, Prefix, Description",
     where='Code == "MON_FORM" && EntityState == "Active"')

2. Extraire l'arborescence des sections :
   get_many_select("Section",
     "Code, Name, OrderBy, Weight, Level, ParentSection.Code, IsTable",
     where='Form.Code == "MON_FORM" && EntityState == "Active"',
     order_by="OrderBy", take=200)

3. Extraire les questions avec types et poids :
   get_many_select("FormQuestion",
     "Code, Question, Reference, Order, Weight, QuestionType, IsRequired,
      Formula, ConditionalDisplay, Section.Code, ResponsePresetCategory.Code",
     where='Form.Code == "MON_FORM" && EntityState == "Active"',
     order_by="Order", take=500)

4. Extraire les reponses possibles :
   get_many_select("FormAnswer",
     "Code, Name, Order, ResponseWeight, IsDefault, Question.Code",
     where='Question.Form.Code == "MON_FORM"', take=1000)

5. Extraire les formules de calcul :
   get_many_select("FormFormula",
     "Code, Formula, LocalizedDescription, Section.Code",
     where='Form.Code == "MON_FORM" && EntityState == "Active"', take=50)

6. Generer le document structure :
   - Page de garde : nom, code, version, description
   - Table des matieres basee sur l'arborescence des sections
   - Pour chaque section : titre, poids, questions
   - Pour chaque question : texte, type, obligatoire, reponses possibles, poids
   - Annexe : formules de calcul
   - Utiliser les skills docx ou pptx selon le format souhaite
```

---

## Workflow 11 — Comparaison de versions d'un formulaire

Objectif : Identifier les differences entre deux versions d'un meme formulaire (template vs deploye, ou version N vs version N+1).

```
1. Identifier les versions disponibles :
   get_many_select("Form", "Code, Name, Version, Description",
     where='ParentCode == "MON_FORM" || Code == "MON_FORM"',
     order_by="Version", take=20)

2. Pour chaque version, extraire :
   - Sections : Code, Name, Level, OrderBy
   - Questions : Code, Question, QuestionType, Weight, IsRequired
   - Reponses : Code, Name, ResponseWeight, Question.Code

3. Comparer en produisant un rapport de differences :
   - Sections ajoutees/supprimees/modifiees
   - Questions ajoutees/supprimees/modifiees (texte, type, poids)
   - Reponses ajoutees/supprimees/modifiees (texte, poids)
   - Formules ajoutees/modifiees

4. Generer un tableau de diff :

   | Element | Version N | Version N+1 | Changement |
   |---------|-----------|-------------|------------|
   | Section ESRSS1>03 | "Exigences" | "Exigences de publication" | Renomme |
   | Question Q012 | (absent) | "Nombre de CDD" | Ajoute |
   | Reponse Q001-NA | Poids: -1000 | (supprime) | Supprime |
```

**Cas d'usage typiques :**
- Recette : verifier que le deploiement n'a pas altere le formulaire
- Audit : tracer les evolutions entre versions
- Migration : comparer avant/apres un import massif

---

## Workflow 12 — Orchestration d'import multi-fichiers

Objectif : Importer les 3 fichiers (Section, Question, Answer) de maniere coordonnee avec validation entre chaque etape.

**Consulter `references/form-module.md` sections Validation pre-import et Guide des erreurs** pour les regles detaillees.

```
ETAPE 0 — PREPARATION
   Decouvrir les langues actives :
     get_many_select("Culture", "CultureCode, IsDefault",
       where='EntityState == "Active"', take=20)
   Verifier le formulaire cible (template, pas versionne) :
     get_many_select("Form", "Code, Name, Version, TemplateId",
       where='Code == "MON_FORM" && EntityState == "Active"')

ETAPE 1 — VALIDATION PRE-IMPORT
   Pour chaque fichier, executer la checklist de validation :
   □ Structure du fichier (colonnes, en-tetes)
   □ Unicite des codes (dans le fichier ET en base)
   □ Coherence des references FK
   □ Coherence croisee entre les 3 fichiers
   → Si erreurs : corriger AVANT de continuer

ETAPE 2 — IMPORT SECTIONS
   Importer le fichier 1 (Section)
   Verifier le resultat :
     get_many_select("Section", "Code, Name, Level, ParentSection.Code",
       where='Form.Code == "MON_FORM" && EntityState == "Active"',
       order_by="OrderBy", take=200)
   Verifier les erreurs :
     get_app_errors(take=5)

ETAPE 3 — IMPORT QUESTIONS
   Importer le fichier 2 (FormQuestion)
   Verifier le resultat :
     get_many_select("FormQuestion", "Code, Question, QuestionType, Section.Code",
       where='Form.Code == "MON_FORM" && EntityState == "Active"',
       order_by="Order", take=500)
   Verifier les erreurs :
     get_app_errors(take=5)

ETAPE 4 — IMPORT REPONSES
   Importer le fichier 3 (FormAnswer)
   Verifier le resultat :
     get_many_select("FormAnswer", "Code, Name, ResponseWeight, Question.Code",
       where='Question.Form.Code == "MON_FORM"', take=1000)

ETAPE 5 — VALIDATION POST-IMPORT
   Compter les elements importes :
     get_entities_count("Section", where='Form.Code == "MON_FORM" && EntityState == "Active"')
     get_entities_count("FormQuestion", where='Form.Code == "MON_FORM" && EntityState == "Active"')
   Verifier la coherence :
   □ Toutes les questions ont une section parente
   □ Toutes les reponses de type pondéré ont un ResponseWeight
   □ L'arborescence des sections est correcte (spInitTreeSectionSingle execute)
```

---

## Workflow 13 — Analyse des permissions et documentation par role

Objectif : Produire une matrice de permissions detaillee pour une entite, avec noms metier des champs, regroupes par DynamicSection, pour documentation ou recette.

```
ETAPE 1 — RECUPERER LES PERMISSIONS BRUTES (DynamicFieldRole)
  get_many_select("DynamicFieldRole",
    "DynamicField.PropertyName, RoleInModule.Name, CanRead, RoleActionForAdd, RoleActionForEdit, Table, Filter",
    where='DynamicField.EntityName == "MonEntite" && EntityState == "Active"', take=200)
  → Retourne N enregistrements (1 par champ × role)
  → Extraire la liste des roles distincts (RoleInModule.Name)

  Valeurs de reference :
  | Champ | Valeurs | Signification |
  |-------|---------|---------------|
  | CanRead | true/false | Visibilite en lecture |
  | RoleActionForAdd | 0=Inactive, 1=Active, 2=ActiveAndHidden, 3=ReadOnly | Comportement en creation |
  | RoleActionForEdit | 0=Inactive, 1=Active, 2=ActiveAndHidden, 3=ReadOnly | Comportement en edition |
  | Table | 0=Inactive, 1=Display, 2=Displayable | Visibilite en liste |
  | Filter | true/false | Disponible comme filtre |

ETAPE 2 — RECUPERER LES NOMS METIER (labels localises)
  Methode 1 — Via get_entity_fields (recommande, retourne LocalizedNames) :
    get_entity_fields("MonEntite")
    → Chaque champ a un dict LocalizedNames avec cles par CultureCode ("fr-FR", "en-US")
    → Utiliser LocalizedNames["fr-FR"] pour le label metier francais

  Methode 2 — Via DynamicFieldName (si get_entity_fields non disponible) :
    get_many_select("DynamicFieldName",
      "Name, ColumnName, Tooltip, DynamicField.PropertyName",
      where='DynamicField.EntityName == "MonEntite" && CultureCode == "fr-FR" && EntityState == "Active"',
      take=100)
    → Name = label metier, ColumnName = en-tete de colonne, Tooltip = aide contextuelle

ETAPE 3 — RECUPERER LES SECTIONS DE FORMULAIRE (DynamicSection)
  get_many_select("DynamicSection",
    "Code, LocalizedName, OrderBy",
    where='DynamicSettings.EntityName == "MonEntite" && EntityState == "Active"',
    order_by="OrderBy", take=20)
  → Retourne les sections avec leur nom et ordre d'affichage

  ⚠ LIMITATION IMPORTANTE : Le mapping champ → section n'est PAS requetable via l'API.
  - La relation DynamicField ↔ DynamicSection est many-to-many via la table de jointure
    `DynamicSectionsFields` (NHibernate HasManyToMany dans DynamicSectionMap.cs)
  - Cette table de jointure n'est pas exposee comme entite dans l'API V2
  - DynamicField n'a pas de FK directe vers DynamicSection
  - DynamicField.OrderBy existe dans le modele C# mais n'est pas accessible via get_many_select
  → Workaround : Inferer le mapping champ → section par connaissance du domaine metier
    ou par lecture du code source C# si disponible

ETAPE 4 — GENERER LE DOCUMENT DE PERMISSIONS
  Utiliser la skill docx pour produire un document Word structure :
  - Page de garde : entite, date, source des donnees
  - Synthese par role : nombre de champs accessibles par permission
  - Matrice croisee : tableau champs × roles avec couleurs par niveau d'acces
  - Detail par role : champs groupes par section avec separateurs visuels
  - Legende : explication des niveaux (Inactive/Active/ActiveAndHidden/ReadOnly)

  Structure du tableau matrice :
  | Section / Champ technique | Label metier | Role 1 | Role 2 | ... |
  |--------------------------|-------------|--------|--------|-----|
  | ▶ Section Formulaire      |             |        |        |     |
  | FormDynamic               | Formulaire  | R/W/W  | R/W/RO |     |
  | ▶ Section Principale      |             |        |        |     |
  | Name                      | Intitule    | R/W/W  | R/W/W  |     |

  Format cellule : Read/Add/Edit (ex: "R/W/W" = CanRead + Active en ajout + Active en edition)
```

**Entites cles :**
- `DynamicFieldRole` : Permissions (CanRead, RoleActionForAdd/Edit, Table, Filter)
- `DynamicField` : Champs (PropertyName, EntityName) — select limite (pas d'OrderBy)
- `DynamicFieldName` : Labels localises (Name, ColumnName, Tooltip, CultureCode)
- `DynamicSection` : Sections de formulaire (Code, LocalizedName, OrderBy) — many-to-many avec DynamicField via `DynamicSectionsFields`
- `RoleInModule` : Roles par module (Name, Code, DynamicModule.Code)

---

## Exemples rapides

```python
# Lister toutes les entites (decouvrir les entites de l'instance connectee)
get_all_entities()

# Champs d'une entite
get_entity_fields("Organization")

# Donnees avec FK (adapter aux entites de l'application connectee)
get_entity_sample_data("MonEntite", "Code, Name, Organization.Name", limit=10)

# Requete filtree
get_many_select("MonEntite", "Name, Code, Organization.Name",
  where='EntityState == "Active"', order_by="Name", take=20)

# Compter
get_entities_count("MonEntite", where='EntityState == "Active"')

# Code injecte C#
get_many_select("DynamicSettings",
  "EntityName, ValidationCreateExpression, PostCreateExpression",
  where='EntityName == "MonEntite"')

# Fonctions reutilisables
get_many_select("DynamicFunction", "Name, Code, Description, CodeFunction",
  where='EntityState == "Active" && Name == "MaFonction"')

# Conditionnalites
get_many_select("Conditionality", "Name, Code, Condition",
  where='EntityName == "MonEntite" && EntityState == "Active"', take=20)

# Workflow
get_many_select("EntityWorkflow", "Name, EntityName",
  where='EntityName == "MonEntite" && EntityState == "Active"')
get_many_select("EntityWorkflowStatus", "Code, LocalizedName",
  where='EntityWorkflow.EntityName == "MonEntite" && EntityState == "Active"', take=20)

# Erreurs recentes
get_app_errors(take=5)

# Formulaires et import (module Form)
get_many_select("Form", "Code, Name, Version, Prefix",
  where='EntityState == "Active"', take=50)
get_many_select("FormQuestionType", "Code, Name",
  where='EntityState == "Active"', take=50)

# URL d'une entite
get_entity_url("MonEntite", entity_id="GUID-ICI", url_action="Edit")
```

Note : Les exemples utilisent `MonEntite` comme placeholder. Les entites metier varient selon l'application (voir regle 11). Commencer par `get_all_entities()` pour decouvrir les entites disponibles.
