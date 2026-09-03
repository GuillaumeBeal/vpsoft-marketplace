---
name: vpsoft-config
version: 1.1.4
description: >-
  Configurer VPSoft (10.5) via le MCP : tables/champs dynamiques (expression C#, formule SQL,
  reverse-link, unité, arbre + niveaux), visibilité, labels, import/export, rendu, menus par profil,
  données & liens, indicateurs, dashboards, workflows (transitions), VISUELS de liste (cards/widgets,
  carte KPI cliquable qui préfiltre), confidentialité, PDF, alertes, aides. Couvre le CODE DYNAMIQUE
  PROPRE au framework VP (C# & JS) : DynamicFunction, DynamicPage/Widget/PageField, code global,
  slots Content*, rendu listes (jTable), DynamicButton, DynamicBatch, expressions d'entité — et les
  CONDITIONNALITÉS & BUSINESS RULES NO-CODE (Hide/Show/ReadOnly/Mandatory/Validation/Filter/
  Affectation). À utiliser dès qu'on parle de VPSoft, FormBuilder, AGL, DynamicSettings/DynamicField,
  table/champ/formulaire/section/menu/module/rôle/permission/indicateur/dashboard/workflow/arbre,
  page dynamique/widget/visuel/card/code global/bouton/batch/PDF/alerte/aide/confidentialité,
  conditionnalité/business rule, ou d'incident VPSoft via MCP.
---

# VPSoft — Configuration via MCP (DynamicFunctions)

Méthode éprouvée pour configurer VPSoft **programmatiquement** via le MCP VPSoft (outils
`mcp__…__*` : `select_app`, `invoke_vpsoft_function`, `create_entity`, `update_entity_patch`,
`get_many_select`, `get_entities_count`, `get_app_errors`, `get_entity_fields`, `list_available_apps`).

Le principe : on écrit des **`DynamicFunction`** (code C# stocké en base, compilé à chaud) qui appellent
les services internes (`IServiceManager`/`IRepositoryManager`), puis on les **invoque** via le MCP.

> **Docs détaillées (à lire pour le détail/preuves file:line) :**
> - `references/MCP-DynamicFunctions-FormBuilder.md` — tables, champs, droits, visibilité, sections, rendu,
>   unités, expression/formule, reverse-link, données, références (catalogue complet + code).
> - `references/MCP-Indicators-Dashboards.md` — **méthode VALIDÉE** indicateurs, dashboards, widgets,
>   workflow dynamique : code des fonctions, preuves source file:line, IDs réels, résultats testés.
> - `references/BOOTSTRAP.md` — **autonomie** : (re)créer toutes les fonctions sur un nouvel environnement
>   (seed `McpCreateFunction` à créer manuellement + code complet des helpers récents + procédure).
> - `references/vpsoft-source-extracts.md` — **digest synthétisé du code source VPSoft** (enums + logique
>   décisive des méthodes citées en `fichier:ligne`), pour comprendre le *pourquoi* sans le dépôt.
> - `references/PORT-DynamicPages-AGLX.md` — **VALIDÉ 2026-09** : **porter du code de DynamicPage entre
>   instances**. Méthode PRÉFÉRÉE (`McpUpdateDynamicPageCode` en texte brut appelée depuis le navigateur),
>   route native **AGLX** testée (API export/import, granularité module vs import sélectif jusqu'à la
>   propriété, 8,4 Mo/module, aucun contrôle de version), **anti-pattern des chunks base64 recopiés par un
>   LLM**, piège du boot DOM des pages pleines, adaptations de champs 10.4→10.5.
> - `references/CODE-DynamicPages-VPFramework.md` — **code dynamique PROPRE** : surface d'API du
>   framework **VP C# & VP JS**, contrat EXACT de chaque expression d'entité (variables en scope,
>   méthode générée), DynamicPage/Widget/PageField (modèle + rendu + création ✅), code global &
>   code global module, slots Content*, rendu custom des listes (jTable), DynamicButton/Batch/Class/Const.
> - `references/NOCODE-Conditionality-BusinessRules.md` — **conditionnalités & business rules no-code** :
>   modèle BusinessRule+RuleTree, JSON des arbres (exemples réels), opérateurs/roots/fonctions,
>   fonctions `McpCreateBusinessRule`/`McpGetRuleTree`/`McpDeleteBusinessRule` ✅ roundtrip validé.
> - `references/CONFIG-TableExtras.md` — **onglets avancés d'une table** : VISUELS (cards/widgets
>   avant-après liste + pattern **carte KPI cliquable qui préfiltre**), CONFIDENTIALITÉ (table +
>   enregistrement), modèles PDF, ALERTES (DataAlert + pipeline), AIDES en ligne — fonctions ✅ testées.

---

## 1. Sécurité serveur & démarrage (IMPÉRATIF)

- **Re-sélectionner l'app à CHAQUE tour** : `select_app("<key>")` (ex. `demo-vesta-10-5`).
  La sélection **se réinitialise entre les tours** (revient sur l'app par défaut) → sinon les appels
  partent sur la mauvaise app (erreurs 500 / HTML).
- **Serveur fragile** : faire des appels **séquentiels et légers**. Éviter le parallélisme massif et
  les `get_many_select` larges (filtrer étroit, peu de champs). En cas de timeouts/`connection aborted`,
  **attendre puis réessayer** ; `list_available_apps` répond même serveur applicatif KO (test de vivacité).
- **Diagnostiquer un 500** : `get_app_errors(search, take:5)` (table AppLog).
- ⚠️ **Les erreurs de type LISTENER ne sont PAS dans AppLog.** `AglExpressionListener` exécute les
  `Listener{Create|Edit}Expression` dans un **scope/session/transaction isolés** et **catch + rollback**
  (`AglExpressionListener.cs:119`) → l'exception n'avorte pas la requête et **n'apparaît pas** dans
  `get_app_errors` (elle part dans le log texte / `DynamicExpressionLogger`). Pour un bug de listener,
  se fier au **log texte applicatif**, pas à AppLog. (cf. §10)

## 2. Anatomie d'une DynamicFunction

- **Tous les paramètres sont `string`** → parser dans le corps (`int.Parse`, `Guid.Parse`,
  `== "true"`, `Split(',')`, `Split('|')`).
- **Toujours un `try/catch` interne** renvoyant `ex.Message` + `ex.GetType().FullName` + `inner` :
  le wrapper **avale les exceptions** (sinon retour `null` muet, impossible à diagnostiquer).
- Résolution des dépendances :
  - `var sm = AppDependencyResolver.GetService<IServiceManager>();`
  - `var rm = AppDependencyResolver.GetService<IRepositoryManager>();` (`using VPSoft.Domain.Repositories;`)
  - `var session = NHSessionHelper.GetCurrentSession();` (`using VPSoft.Domain.Helpers;`) — pour
    manipuler des entités dynamiques en direct (Criteria + reflection) quand l'API REST échoue.
- **Créer une fonction** : `McpCreateFunction(name, parameters, returnValue, codeUsing, codeFunction, isAsync, moduleId, folderId)`.
  `returnValue`=`System.Object`, `isAsync`=`false` (sauf build). Contrôle d'unicité du nom (idempotent).
  - ⚠️ **RÈGLE : toutes les fonctions `Mcp*` doivent vivre dans le dossier « API MCP » (module System =
    Administration), JAMAIS dans le dossier du module métier.** Donc **passer `folderId` = l'Id du dossier
    « API MCP »** (le `folderId` est prioritaire sur le `moduleId` dans le seed) ; **ne PAS** passer un
    `moduleId` métier seul (ça range la fonction dans la racine de ce module). L'Id du dossier est
    **propre à chaque base** → le résoudre via une fonction Mcp existante :
    `get_many_select("DynamicFunction","DynamicFolder.Id","Name == \"McpCreateFunction\"")`.
    Pour rapatrier des fonctions mal rangées : **`McpMoveFunctionsToFolder(namesCsv, folderId)`**.
- **Config vs schéma** : changements de **schéma** (table/colonne/relation, 1ʳᵉ unité) → `McpBuild()` ensuite.
  La **config** (droits, libellés, rendu, sections, menus, indicateurs, lecture seule, triggers) prend
  effet **immédiatement** (cache `[EntityCache]`), **sans rebuild**. (Faire `F5` côté UI.)

## 3. Catalogue des fonctions `Mcp*` (déjà déployées)

| Fonction | Rôle | Rebuild |
|----------|------|---------|
| `McpCreateTable` | Créer une table (entité) | Oui |
| `McpCreateField` | Champ standard (String/Int/Decimal/Date/Bool/Entity…) | Oui |
| `McpCreateEnumField` | Champ enum (liste de valeurs) | Oui |
| `McpCreateExpressionField` + `McpSetFieldExpression` | Champ **expression C#** (corps avec `return`) | Oui |
| `McpCreateFormulaField` + `McpSetFieldFormula` | Champ **formule SQL** (déconseillé) | Oui |
| `McpCreateReverseLink` | Champ entité + propriété inverse (collection ↔ référence) | Oui |
| `McpBuild` | Matérialiser le schéma | — |
| `McpGrantTablePermission` | `RolePermission` d'entité (CanSee/Add/Edit/Delete) | Non |
| `McpSetFieldsApi` | Exposition **API REST** des champs (liste blanche) | Non |
| `McpSetFieldsUiVisibility` | Visibilité **liste/create/edit** (Table/Add/Edit=Active) | Non |
| `McpSetFieldReadOnly` | Champs en **lecture seule** create/edit (Add/Edit=ReadOnly) | Non |
| `McpSetFieldsEditInLine` | **Édition inline** dans la liste (`DynamicFieldRole.EditInLine`) | Non |
| `McpSetExpressionTriggers` | **Triggers** de recalcul live d'un champ expression | Non |
| `McpSetFieldExport` | Exposition **import/export** (liste blanche, reset des autres en NotAvailable) | Non |
| `McpSetFieldsImportExport` | Import/export par **mode** (Both/OnlyExport/NotAvailable), **sans reset** des autres | Non |
| `McpSetFieldRender` | **Rendu** global (`RenderTypeOverrided`, `FieldDisplay`, `DontDisplayLabel`) | Non |
| `McpSetFieldNestedTable` | **Tableau imbriqué** (`RenderType.Table` + entité enfant) | Non |
| `McpSetFieldUnit` | **Unité** sur champ numérique (`FormProperty.Unit`) | **Oui** (régénération requise à chaque pose/changement) |
| `McpCreateUnit` | Créer une **unité de référence** + sa mesure de base (ex. Distance/km) | Non |
| `McpClearFieldUnit` | **Retirer l'unité** d'un champ — restaure l'écriture REST/MCP (cf. §6) | **Oui** |
| `McpSetFieldLabel` | Libellés multilingues (`DynamicFieldName`) | Non |
| `McpSetFieldTexts` | **Label + infobulle + description** (par culture) en un appel | Non |
| `McpAddFieldsToSection` | Placer des champs dans une **section** de formulaire | Non |
| `McpAddTableToMenu` | Faire apparaître une table dans le **menu** (NavMenu) + visibilité rôle | Non |
| `McpSetReference` | **Lier** 2 enregistrements par code (NHibernate direct, contourne le bug PATCH) | Non |
| `McpDeleteEntityByCode` | **Supprimer** un enregistrement par son entity code `Code` (NHibernate direct) | Non |
| `McpSetDashboardOwner` | **Transférer la propriété** d'un dashboard (`Dashboard.CreatedUser`) par Id — débloque l'édition (cf. §9) | Non |
| `McpDiagMenuModel` | **Audit menu** : items d'un module + leurs profils (`NavMenu.RolePermissions`, non requêtable autrement) | — |
| `McpSetMenuRoles` | **Menu PAR PROFIL** : pose l'ensemble des `RoleInModule` (Name/Code) sur le `NavMenu` d'une table | Non |
| `McpGrantTablePermissionForRole` | Permission d'entité (`CanSee/Add/Edit/Delete`) **pour un profil cible** | Non |
| `McpSetFieldsUiVisibilityForRole` | Visibilité liste/create/edit **pour un profil cible** | Non |
| `McpSetEntityCode` | **Code métier / business rules** sur l'entité (`DynamicSettings`, slots C# serveur + front) | **Oui** (slots C# serveur) |
| `McpSetWorkflowActionCode` | **Code de transition** workflow (`EntityWorkflowAction.ValidationExpression`/`InjectionExpression`) | **Oui** |
| `McpCreateFunction` | Créer de nouvelles DynamicFunctions (⚠️ toujours `folderId` = dossier « API MCP », pas le module métier) | — |
| `McpUpdateFunctionCode` | **Mettre à jour le code** d'une DynamicFunction (PATCH générique = 500 → GetSingle+Edit) | — |
| `McpMoveFunctionsToFolder` | **Déplacer des `Mcp*` vers le dossier « API MCP »** (`namesCsv, folderId`) — corrige les fonctions rangées par erreur dans un module métier | — |
| `McpCreateDynamicPage` | **Page dynamique / widget / page-champ** (`DynamicPageBaseService.Create`, nom multilingue) | Non |
| `McpRenderDynamicPage` | **Rendre une DynamicPage sans navigateur** (Razor compilé + CSS + JS, mode preview) | — |
| `McpAttachFieldPage` | **Attacher une page-champ** aux slots `DynamicPageField{Add\|Edit\|Details}` d'un champ (+`RenderType.DynamicPage`) ; `actions="clear"` pour détacher | Non |
| `McpSetTreeLevels` | **Niveaux d'une table ARBRE** (`TreeConf`+`TreeConfLevel`, nombre OU noms/couleurs, idempotent, max 10) — ⚠️ `McpCreateTable isTree` n'en crée AUCUN | Non |
| `McpCreateDynamicButton` | **Bouton dynamique** (position Form/Section/Subsection/Table/TableHeader, nom localisé) | Non |
| `McpSetButtonCode` | **Code d'un bouton** : `DynamicExpression` (C# serveur) + `JSPreCall`/`JSCallBack` (JS) | Non |
| `McpDeleteDynamicButton` | Supprimer un bouton par `Code` | Non |
| `McpCreateBatch` | **DynamicBatch** (CodeBatch/ClassBatch/références, Name unique) | Non |
| `McpRunBatch` | **Exécuter un batch** (wrapper `MainVP()`+`Context`) → returnMessage + logs | — |
| `McpCreateBusinessRule` | **Business rule no-code** (TOUS les kinds : Hide/Show/ReadOnly/Mandatory/Validation/Filter/Affectation/Activation/Alert/Propagation) + arbre d'activation | Non |
| `McpSetBusinessRuleExtras` | **Compléments par type** : `actionTree` (Filter), `affectationSpec` (⚠️ `ParameterValueDto`, pas un arbre), `validationMessageKey`, `mandatoryMessageKey`, `priority`, `dynamicFieldId` | Non |
| `McpGetRuleTree` | **Lire un arbre de règle** (JSON `RuleNodeDto`) — seul accès aux arbres (pas `get_many_select`) | — |
| `McpDeleteRuleTree` | Supprimer un arbre isolé (orphelin, RuleSet) | — |
| `McpDeleteBusinessRule` | **Supprimer** une business rule + ses arbres (`DeleteWithDependencies`) | Non |
| `McpAddTableWidget` / `McpDeleteTableWidget` | **VISUELS** : rangée de cards (texte/widget) avant/après une liste, PAR RÔLE (`DynamicSectionTemplate`) | Non |
| `McpSetColumnWidth` | **Largeur des cards VISUELS** : passe toutes les colonnes d'une section à un `ClassWidth` (ex. `"col"` pour N cartes sur une ligne — corrige le retour 3+1, cf. CONFIG-TableExtras §1.3) | Non |
| `McpSetSectionRole` | **Rattacher une section VISUELS au BON `RoleInModule`** (par module) — corrige une section créée sous un rôle homonyme du mauvais module (visible liste user mais introuvable dans l'admin Visuels, cf. CONFIG-TableExtras §1.3) | Non |
| `McpListModuleFolders` | **Lister les dossiers d'un module** (`DynamicFolder` via NHibernate — le `where:"DynamicModule.Id==…"` côté API V2 échoue « SQL not available ») | — |
| `McpCreateFolderMovePages` | **Créer un dossier nommé dans un module + y ranger des pages/widgets** (les widgets créés via MCP atterrissent dans `System/API MCP` → les ranger dans le bon module ; nom localisé via `CultureParameterService.SaveOrUpdate`) | Non |
| `McpSetPageName` | **Renommer une page/widget** (`pageCode, value`) — pose le `LocalizedName` (CultureParameter) sur **toutes les cultures** en 1 appel (≠ `McpUpsertLocalizedName` qui est par culture) | Non |
| `McpSetTableConfidentiality` | **Confidentialité TABLE** (`IsConfidentialMaster` + maître) | Non |
| `McpSetRecordConfidentiality` | **Confidentialité ENREGISTREMENT** (`IsConfidential` + users/roles, NH direct) | Non |
| `McpCreateWorkflowAction` / `McpDeleteWorkflowAction` | **Transition de workflow** (statut source→cible, rôles, couleur, nom localisé) | Non |
| `McpRunWorkflowAction` | **Exécuter une transition** sur un enregistrement (statut + historique + injection) | — |
| `McpCreatePdfModel` | **Modèle PDF** d'entité (`EntityPDFModel` + sections Razor/HTML) | Non |
| `McpCreateDataAlert` | **Alerte** (`DataAlert` notification, rôles destinataires, localisée — créer Inactive puis activer) | Non |
| `McpCreateOnlineHelp` | **Aide en ligne** (`HelpOnline` : 4 contextes localisés + rôles) | Non |
| `McpSetFieldsFilter` | **Filtres de liste** : pose `DynamicFieldRole.Filter=true` (colonnes filtrables dans le bandeau) par liste blanche/rôle (modèle `McpSetFieldsEditInLine`) | Non |
| `McpSetNavMenuIconOrderBulk` | **Icône + ordre** des entrées de menu (`NavMenu.Icon`/`OrderBy`) par Id, en lot — JSON `[{"id","icon","order"}]` (clé absente = inchangée) | Non |
| `McpAddPageToMenu` | **Ajouter une DynamicPage au menu** d'un module (`NavMenuDynamicPage` via `NavMenuService.ActivateOrCreate`, `MenuType.DynamicPage`=1) + icône/ordre/rôles/nom localisé | Non |
| `McpSetDynamicPageRoles` | **Rôles d'accès** d'une DynamicPage/widget (`RoleInModules`) — ⚠️ sinon `UnauthorizedAccessException` au chargement hors preview (`McpCreateDynamicPage` ne les pose PAS) | Non |
| `McpUpdateDynamicPageCode` | ⭐ **VOIE PRÉFÉRÉE — MAJ du code d'une DynamicPage en TEXTE BRUT**, à appeler **depuis le navigateur** avec le contenu de fichiers locaux (cf. `references/PORT-DynamicPages-AGLX.md` §2) ; ⚠️ `DynamicPageCodeViewModel.GitFileBasePath` est `required` → passer par `GetCode(id)`, pas un `new{}` ; param vide = conservé | Non |
| `McpSetDynamicPageJsB64` | ⚠️ **NE PLUS UTILISER pour un transfert piloté par un LLM** (cf. §6) — préférer `McpUpdateDynamicPageCode` depuis le navigateur. Injecte un gros JS via **base64** (params `code,b64,reset,finalize` : accumule les chunks puis décode `Convert.FromBase64String`) — contourne l'échappement JSON des gros blocs | Non |
| `BdgCreate` (générique) | **Créer une entité dynamique AVEC références** (params `entityName, fieldsJson` ; réflexion + NHibernate : enum→`Enum.ToObject(int)`, réf→chargée par `Code`, décimal/date castés, Code auto, audit) — à appeler côté page via `VP.Functions.Invoke` ⚠️ car `VP.Entities.Create` (endpoint V2) ne sait PAS lier les références | Non |

> **Autres fonctions présentes en base** (variantes/diagnostic, non détaillées ici) : `McpBuildAsync`/`McpBuildStatus`
> (build non bloquant), `McpCreateDrillDownIndicator`/`McpCreateMatrixIndicator`/`McpSetIndicatorMeasureUnit`/
> `McpResolveDashboardId`/`McpRecalcExpressionFields`, `McpCreateReferenceField`, `McpCreateSection`, `McpSetFieldDescription`.
> **Obsolètes — NE PAS utiliser** (remplacées) : `McpExposeFieldsToApi`, `McpGrantFieldApiAccess`,
> `McpGrantFieldApiAccessDbg` (→ `McpSetFieldsApi`), `McpWhoAmI` (debug).

### Indicateurs / Dashboards / Workflow (détail : `references/MCP-Indicators-Dashboards.md`)

| Fonction | Rôle | Rebuild |
|----------|------|---------|
| `McpCreateEntityWorkflow` | Workflow dynamique (`EntityWorkflow` + 3 statuts À soumettre/Approuvé/Refusé) — **l'ACTIVE** | Non |
| `McpSetWorkflowActive` | **Activer/désactiver** le workflow d'une entité (`EntityState`) — masquer sans supprimer, réversible | Non |
| `McpCreateIndicator` | Indicateur agrégé mono-mesure (SUM/AVG/COUNT… + dimension de regroupement) | Non |
| `McpCreateMultiMeasureIndicator` | Indicateur à plusieurs mesures (`measuresJson`) | Non |
| `McpCreateTypedIndicator` | Indicateur de **n'importe quel type de graphe** (`chartType`, cf. enum `ChartRepresentation`) | Non |
| `McpSetIndicatorChartType` | Changer le **type de graphe** d'un indicateur existant (PATCH `Indicator` = 500 → GetSingle+Edit) | Non |
| `McpCreateDashboard` | Crée `DynamicDashboard` + `DashCollection` + `Dashboard` (conteneur à widgets) | Non |
| `McpAddIndicatorWidget` | Ajoute un **widget indicateur** (`DashboardReportWidget → Indicator`) à un dashboard | Non |
| `McpDiagReportData` | **Diagnostic** : rend le `ReportViewModel` réel d'un indicateur (headers+data) sans navigateur | — |

> ⚠️ **On appelle les fonctions par NAME**, jamais par GUID : `invoke_vpsoft_function("McpXxx", [...])`.
> Les `DynamicFunction.Id` (GUID) sont **propres à chaque base** (non portables d'une app à l'autre) →
> les tableaux d'IDs des références sont de la **traçabilité démo uniquement**. Vérifier l'existence par Name
> (`get_many_select("DynamicFunction","Name","Name == \"…\"")`) ; recréer via `McpCreateFunction` si absent
> (idempotent sur le nom). Quand un `functionId` est requis en paramètre (ex. `McpUpdateFunctionCode`),
> **le résoudre par Name** : `get_many_select("DynamicFunction","Id","Name == \"McpXxx\"")`.

## 4. Les couches de visibilité (modèle mental clé)

Rendre un objet « visible » = plusieurs leviers **indépendants** :

**Pour un CHAMP :**
1. **Écriture API** → `DynamicFieldRole.Api` (`McpSetFieldsApi`). Sans ça, l'API ignore le champ silencieusement.
2. **Vue liste + éligibilité create/edit** → `DynamicFieldRole.Table`/`RoleActionForAdd`/`RoleActionForEdit`
   (`McpSetFieldsUiVisibility` ; `McpSetFieldReadOnly` pour Add/Edit=ReadOnly).
3. **Présence dans le formulaire** → la **section** (`DynamicSection.DynamicSectionPropertyNames`,
   `McpAddFieldsToSection`). Un champ sans section n'apparaît pas au formulaire même si visible en liste.
4. **Import/export** → `DynamicFieldRole.ImportExport` (`McpSetFieldExport`).

**Pour une TABLE dans la navigation :**
1. **Permission d'entité** → `RolePermission.CanSee` (+Add/Edit/Delete) via `McpGrantTablePermission`.
   Sans elle, l'API renvoie **HTTP 500**.
2. **Item de menu** → **`NavMenu`** (sous-classe `NavMenuDynamicSettings`) rattaché au `DynamicModule`,
   `IsActiveUser`/`IsActiveSetting`, `OrderBy`, `Icon`, et **`RolePermissions` (ISet<RoleInModule>)** =
   quels profils voient l'item. Créé par `McpAddTableToMenu`.
   - Service : `NavMenuService.ActivateOrCreate(NavMenuCreateOrActivateFormModel{ Name, EntityId(=DynamicSettings.Id),
     DynamicModuleId, Type=MenuType.EntityDynamic(2), IsForSettings, Order })` **ne pose PAS** la visibilité rôle
     → poser ensuite `navMenu.RolePermissions` (ce que fait `McpAddTableToMenu`).
   - Arbre par utilisateur construit par `NavMenuService.GetNavMenus` → `DynamicModuleService.GetNavMenusItemsFromModuleId`,
     filtré sur `user.RoleInModules` vs `NavMenu.RolePermissions` (+ entité `CanSee`).
   - ⚠️ **Gap MCP** : une table créée via `McpCreateTable` n'a **pas** de NavMenu (contrairement à l'UI
     `AddDynamicEntity`) → il faut appeler `McpAddTableToMenu` pour la voir dans le menu.

> **⚠️ Mono-profil vs PAR PROFIL.** Par défaut, `McpAddTableToMenu`, `McpGrantTablePermission` et les
> fonctions de visibilité champ (`McpSetFieldsApi/UiVisibility/ReadOnly/Export/EditInLine`) résolvent le
> rôle via **l'utilisateur MCP courant** → elles ne configurent **qu'un seul profil**. Pour gérer
> **plusieurs profils / un profil ciblé** (ex. `Reader`, `Contributeur`), utiliser les variantes qui
> prennent un **`RoleInModule` par Name/Code** dans le module :
> - **Menu** : `McpSetMenuRoles(entity, moduleId, "VPW Admin,Reader")` (ensemble exact des profils sur le NavMenu).
> - **Permission d'entité** : `McpGrantTablePermissionForRole(entity, moduleId, "Reader")`.
> - **Visibilité champs** : `McpSetFieldsUiVisibilityForRole(entity, moduleId, "Reader", "Champs")`.
> - **Audit** : `McpDiagMenuModel(moduleId)` (items + profils), et lister les profils du module via
>   `get_many_select("RoleInModule","Name,Code","DynamicModule.Code == \"<module>\"")`.
> Les profils (`RoleInModule`) doivent **préexister** dans le module ; filtrer DFR par rôle via
> `RoleInModule.Code` (le `RoleInModule.Name` n'est pas résoluble dans le WHERE de `DynamicFieldRole`).

## 5. Recette : rendre une table **pleinement** exploitable + visible

```
0) select_app("demo-vesta-10-5")
1a) McpCreateTable(...) ; McpBuild()        // ⚠️ builder la table AVANT d'ajouter des champs
    // ⚠️ TABLE ARBRE (isTree=true) : appeler AUSSI McpSetTreeLevels(entity, "3" ou
    // [{"name":"Pays","color":"#…"},…]) — la création ne pose AUCUN niveau (arbre inutilisable sinon).
1b) McpCreateField(...) × N ; (expression/formule/reverse-link…) ; McpBuild()   // 2e build
    // McpCreateField/expression/formule EXIGENT que la table (et l'entité cible d'un reverse-link)
    // soit DÉJÀ buildée — elles résolvent le type compilé. Sinon retour null/echec silencieux.
2) McpGrantTablePermission(table, moduleId)                 // couche entité (sinon 500)
3) McpSetFieldsApi(table, moduleId, "Champs,API,…")          // écriture API
4) McpSetFieldsUiVisibility(table, moduleId, "Champs,liste,form")
5) McpAddFieldsToSection(table, "Formulaire", "Champs,form") // présence au formulaire
6) McpSetFieldTexts(table, champ, "fr-FR", label, infobulle, description) × champs
7) McpSetFieldExport(table, moduleId, "Champs,export")       // si import/export voulu
8) McpAddTableToMenu(table, moduleId, order)                 // apparaît dans la navigation
9) (option) McpSetFieldReadOnly / McpSetExpressionTriggers / McpSetFieldRender / McpSetFieldNestedTable
10) Données : create_entity(...) ; liens via "Champ":"<code>" (PAS _Code_) ou McpSetReference
```

## 6. Gotchas catalogués (issus de vrais incidents)

- ⚠️ **NE JAMAIS faire recopier un payload (base64 ou texte) par un LLM/sous-agent** pour remplir un slot de
  code : il perd/ajoute des caractères et réduire les chunks **ne converge pas** (6000 → 2000 → 20 car.,
  **138 appels, ~70 min, jamais abouti** — vécu 2026-09). Les octets doivent aller **disque → navigateur →
  serveur** : `McpUpdateDynamicPageCode` appelée depuis la page via `VP.Functions.Invoke` (**9 slots en 0,5 s**).
  Détail : `references/PORT-DynamicPages-AGLX.md`.
- ⚠️ **`window.VP` est `undefined`** dans une page VPSoft alors que **`VP` nu existe** (binding global du
  bundle) → toujours tester/appeler `VP` nu, sinon faux négatif « VP absent ».
- ⚠️ **`System.IO.Compression` (GZipStream) n'est PAS résolvable** en compilation à chaud des
  DynamicFunction → retour `null` **muet**. Idem un bloc `using (var h = SHA256.Create()){}` (instancier sans `using`).
- ⚠️ **Une page pleine (via menu) exécute son script AVANT que le DOM soit prêt** : un
  `getElementById(...).addEventListener` **non gardé** lève et **annule TOUS les bindings suivants**
  (symptôme : boutons inertes, souvent **sans erreur visible** en console) → envelopper l'init dans un boot
  `DOMContentLoaded` + attente de l'élément clé.
- ⚠️ **Une sonde SQL cross-base (`sys.databases`) dans un prompt de sous-agent est bloquée par le
  classifieur de sécurité** : l'agent meurt et le travail est à relancer. S'en passer.
- **Référence en `create_entity`** : passer le champ **directement = code** (`{"Vehicule":"DEMO-EXPR2"}`),
  **pas** le suffixe `_Code_` (resté `null`).
- **`update_entity_patch` peut renvoyer 500** sur une entité avec **reverse-link** : cause racine = **référence
  circulaire** (collection inverse ↔ référence) → **Mapster** récurse à l'infini (pas de `PreserveReference`).
  Côté JSON c'est gardé, côté Mapster non. **Contournement fiable : `McpSetReference`** (NHibernate direct, sans Mapster).
  (Fix appli recommandé R&D : `TypeAdapterConfig.GlobalSettings.Default.PreserveReference(true)` / `MaxDepth`.)
- **Réassigner un champ d'AUDIT (`CreatedUser`/propriétaire) d'un enregistrement existant** : l'audit ne pose
  `CreatedUser`/`CreatedDate` qu'au **`Save` (insert)**, **jamais à l'`Edit`** (`NHRepository.cs:132-135` —
  `entityAudit.CreatedUser = crtUser` n'est QUE dans `Save`). Donc on peut **réaffecter** `CreatedUser` via
  une DynamicFunction NHibernate : `repo.GetSingle(x=>x.Id==id)` → `e.CreatedUser = rm.UserRepository.LoadReference(uid)`
  → `repo.Edit(e)` (persiste, non réécrasé). `update_entity_patch` sur ces entités = 500 Mapster → toujours
  passer par DynamicFunction. Ex. validé : **`McpSetDashboardOwner`** (transfert de propriété d'un dashboard).
- **Champ expression C#** : `ReferenceLambda` est un **corps de méthode** (doit contenir `return`, ex.
  `"return Kilometrage + KilometrageAnnuel;"`), **pas** un lambda `x => …`. Recalcul **live** seulement si
  `ExpressionTriggerFields` est renseigné (= les champs de la formule).
- **Champ formule SQL** : référence les **colonnes physiques** `FormBuilder_<Prop>` (pas le nom de propriété).
  Approche **déconseillée** vs expression C#.
- **`get_many_select`** : tri = `order_by` (pas `order`) ; propriétés calculées non requêtables
  (`Culture.CultureCode`, pas `CultureCode`) ; **on ne peut pas projeter une collection** (`X.RolePermissions.Id` → 400) ;
  traverser les liens par `Lien.SousChamp`.
- **`&&` dans un `where` MCP peut arriver HTML-encodé** (`&amp;&amp;` ⇒ `Syntax error ';'`) →
  écrire **`and` / `or`** (Dynamic LINQ les accepte). ✅ constaté en conditions réelles.
- **Entités hors `IEntity` non requêtables** via `get_many_select` (ex. `RuleNode`/`RuleClause` des
  business rules → erreur `violates the constraint of type parameter 'TEntity'`) → passer par le
  service dédié dans une DynamicFunction (ex. `McpGetRuleTree`).
- **⚠️ Noms de types ambigus OU hors usings par défaut en compilation à chaud ⇒ retour `null` muet
  (vécu ×2).** Cas avérés : `FormContext` (ambigu → **qualifier** `VPSoft.Domain.Enums.FormContext.None`) ;
  enums `DataAlert*`/`AlertActions` (namespace `VPSoft.Domain.Enums.Notifications` ABSENT des defaults →
  l'ajouter en CodeUsing) ; `PdfPageOrientation` (→ `using EvoPdf;`). Diagnostic : bissecter via
  `McpUpdateFunctionCode` (`return "pong";` puis réintroduire ligne à ligne).
- **`AlertActions` est un `[Flags]`** : None=0, Create=1, Edit=2, CreateAndEdit=3, Delete=4.
- **`DataAlert.LocalizedName` non mappé** (propriété non-virtual) → `get_many_select` le lit `null` ;
  lire/écrire via `CultureParameter` (`ParameterId == <alertId>`).
- **500 « Transaction not connected » (page HTML)** : hoquet transactionnel du serveur — l'écriture est
  rollbackée. Vérifier l'absence d'objet partiel (`get_many_select` par Name) puis **réessayer** (validé).
- **Table ARBRE sans niveaux = inutilisable** : `McpCreateTable(isTree)` ne crée ni `TreeConf` ni
  `TreeConfLevel` → toujours enchaîner `McpSetTreeLevels` (max 10 niveaux, idempotent).
- **Unité (champ) → REBUILD OBLIGATOIRE** : `McpSetFieldUnit(table, champ, unitId)` pose
  `FormProperty.Unit`, mais **toute pose/changement d'unité exige un `McpBuild`** pour être prise en compte
  (le **formatage avec l'unité est généré au build** — champs virtuels « formatés » + mapping `DynamicMeasures`
  dans `ClassBuilder`/`ClassMappingBuilder` ; et `DynamicField` est `[EntityCache]`, l'édition via repository
  ne rafraîchit pas le cache). ⚠️ Le `rebuildRequired` renvoyé par `McpSetFieldUnit` ne reflète QUE le besoin
  **schéma** (relation `DynamicMeasures`, vrai seulement la 1ʳᵉ unité de l'entité) — **lancer `McpBuild`
  systématiquement** après un changement d'unité, même si `rebuildRequired=false`.
- **⚠️ Unité ⇒ champ NON-ÉCRIVABLE via REST/MCP (validé en conditions réelles).** Poser une unité sur un
  champ numérique fait passer sa valeur par le mécanisme `DynamicMeasures` que l'importeur REST
  (`create_entity`/import) **n'alimente pas** → la valeur est stockée **`null`** (et tout champ
  expression/formule qui en dépend devient null aussi). **Le formulaire UI fonctionne** (le measure y est
  géré) ; seul le chemin **MCP/REST** est touché. → Si tu injectes des données via MCP : **poser les unités
  APRÈS le chargement**, ou ne pas mettre d'unité sur les champs peuplés par API. **Récupération** :
  `McpClearFieldUnit(table, champ)` (retire l'unité) puis `McpBuild` → l'écriture REST est restaurée.
- **⚠️ Builder la table AVANT d'ajouter des champs.** `McpCreateField`/expression/formule **exigent** que
  la table (et l'entité cible d'un reverse-link) soit **déjà buildée** (elles résolvent le type compilé via
  `ReflectionHelper.GetTypeEntity`). Sinon : retour **`null`/échec silencieux**. Séquence :
  `McpCreateTable → McpBuild → McpCreateField×N → McpBuild`.
- **Boolean** : `McpCreateField` pose désormais `DefaultValue="0"` automatiquement (corrigé) → un champ
  Boolean se crée sans erreur. (Avant le fix : `"DefaultValue doit être 0 ou 1"`.)
- **Unités = DONNÉES DE RÉFÉRENCE** (écran `/System/Settings/Unit`, entités `Unit` + `Measure`, pas du code).
  Une `Unit` = catégorie (`UnitType` : Currency/Length/Surface/Mass/Volume/Energy/Time/Custom…) + une **mesure
  de base** (`Measure` `IsBasicUnit=true`, ex. symbole `km`). ⚠️ **Ne pas se fier au `Name`** (souvent mal
  libellé en démo : l'unité `Code=Length` peut être un **pourcentage** mal codé, défaut `%`). Choisir par
  `UnitType`/contenu réel. **Aucune unité km n'existe par défaut** → en créer une via **`McpCreateUnit`**
  (`UnitService.CreateUnit(UnitCreateFormModel{Name,Code(unique),UnitTypeString})` + une `Measure` base
  `Symbol="km", Value=1, IsBasicUnit=true, Unit=…`). `CreateUnit` **ne crée PAS** la mesure (sauf type Derived)
  → créer la `Measure` à part (`MeasureService.Create`). `Code` d'unité **unique**.
- **Champ expression — recalcul LIVE** : `McpSetExpressionTriggers(table, champExpr, "ChampsSources")` pose
  `DynamicField.ExpressionTriggerFields` (= les champs de la formule). Sans ça, recalcul **seulement au save**.
- **Champ expression/formule en lecture seule** : `McpSetFieldReadOnly(table, moduleId, "Champs")` met
  `RoleActionForAdd/Edit = ReadOnly(3)` (garde `Table` visible). Règle générale : les champs calculés
  (expression C# / formule SQL) doivent être **lecture seule même en Edit**.
- **Workflow = OPTIONNEL & quasi-impossible à supprimer.** La création de table **n'active aucun workflow** ;
  un workflow n'apparaît que si on appelle explicitement `McpCreateEntityWorkflow` (qui l'**active**,
  `EntityState=Active`). Affichage gaté par : enregistrement avec un `EntityWorkflowStatus` **ET**
  `EntityWorkflow.EntityState==Active`. ⚠️ **Ne JAMAIS supprimer** (cascade statuts/actions/historique,
  risqué/irréversible). Pour masquer : **`McpSetWorkflowActive(entity, "false")`** → `EntityState=Inactive`
  (réversible, rien supprimé). Donc : **ne créer un workflow QUE sur demande explicite.** (F5 côté UI.)

## 7. Repères d'environnement (démo)

| Élément | Valeur (démo) |
|--------|----------------|
| App | `demo-vesta-10-5` (`https://demo.vpwhite.com/Vesta_Demo`) |
| Module/Folder « Migration » (DynamicModule) | `f368e802-3893-437a-8c77-acbd44c89b1a` (sert de `moduleId`) |
| Rôle (RoleInModule) Migration | `3fa980e6-845d-4cca-b771-16e170b688f7` |
| Dossier fonctions API MCP | `d019db8c-8b9a-4922-b3d2-d3145638b1b2` |
| Module HÔTE des fonctions `Mcp*` (pour `McpCreateFunction`) | `831cc136-94cc-4e0b-b783-79f4f12442a7` (**≠** module « Migration » ci-dessus) |
| Entité démo | `McpDemoVehicule` (DynamicSettings `8ab3cb38-5502-4a08-b18f-d4857f93926e`) + enfant `McpDemoIntervention` (`e3732dce-…`) |
| Table arbre démo | `McpDemoArbre` (DynamicSettings `9922977a-87e5-4ab9-94f5-e540bce07f27`, 5 niveaux, TreeConf `31fdebec-…`) |
| Exemples vivants | widget `ZZMcpTestWidget`, page-champ `ZZMcpTestPageField`, batch `ZZ_McpTestBatch` (dossier MCP) |
| Exemples vivants (onglets) | visuel `MCP_13f20344` (cards avant liste McpDemoVehicule), aide « Aide - Suivi des véhicules », PDF « ZZ Fiche véhicule (exemple MCP) », alerte `ZZ_MCP_ALERT` (Inactive) |
| Rôle VPW Admin (RoleInModule.Code) | `228B4CB4-50DD-4009-AF65-F542EF6EE3E7` (présent dans 2 modules) |

> Ces IDs sont **propres à la démo**. Pour un autre environnement, les résoudre dynamiquement :
> module/folder via `get_many_select("DynamicSettings","DynamicFolder.DynamicModule.Id", "EntityName == \"…\"")`,
> rôle via le rôle de l'utilisateur MCP courant dans le module.
> ⚠️ **Pour `McpCreateFunction(... moduleId, folderId)`** : ces 2 IDs = là où **ranger la fonction**. **RÈGLE
> utilisateur (impérative) : TOUJOURS ranger les `Mcp*` dans le dossier « API MCP » (module `System` =
> Administration)** → **passer `folderId`** = l'Id de ce dossier (prioritaire sur `moduleId`), **jamais** un
> `moduleId` métier seul (sinon la fonction atterrit dans la racine du module métier — erreur à corriger via
> `McpMoveFunctionsToFolder`). L'Id « API MCP » est **propre à chaque base** → le résoudre depuis une fonction
> existante : `get_many_select("DynamicFunction","DynamicFolder.Id","Name == \"McpCreateFunction\"")`
> (démo `demo-vesta-10-5` : `d019db8c-8b9a-4922-b3d2-d3145638b1b2`).

## 8. Vérification systématique (en base)

Après chaque action, **vérifier via `get_many_select`** (étroit) :
- Schéma : `get_entity_fields(entity)`.
- Visibilité champ : `DynamicFieldRole` filtré `RoleInModule.Id == "<role>" && DynamicField.EntityName == "<table>"`
  → colonnes `Api`, `Table`, `RoleActionForAdd`, `RoleActionForEdit`, `ImportExport`.
- Sections : `DynamicSectionPropertyName` filtré `DynamicSection.DynamicSettings.EntityName == "<table>"`.
- Libellés/infobulles : `DynamicFieldName` filtré `Culture.CultureCode == "fr-FR"`.
- Menu : `get_entities_count("NavMenu", "Name == \"<table>\"")` (0 = pas dans le menu) puis
  `NavMenu` → `MenuType/IsActiveUser/OrderBy/DynamicModule.Code`.
- Données/lien : `get_many_select(table, "…,Ref.Code", "…")`.

## 9. Indicateurs & Dashboards & Workflow dynamique

Méthode **validée et testée** (code des fonctions + preuves source file:line + IDs réels + résultats) dans
`references/MCP-Indicators-Dashboards.md`. Mêmes conventions que FormBuilder (DynamicFunction à paramètres
`string`, try/catch interne, vérif source avant codage, vérif en base, prudence serveur, `select_app` à chaque tour).

### Modèle mental
- **Workflow dynamique** = `EntityWorkflow` (+ 3 `EntityWorkflowStatus` À soumettre/Approuvé/Refusé) via
  `EntityWorkflowService.Create` + `CreateEntityWorkflowDefautStatus`. → `McpCreateEntityWorkflow`.
- **Indicateur** = `IndicatorService.CreateIndicator(IndicatorPreviewFormModel)` : agrégation SQL sur une
  entité. Composé de `IndicatorData` (les **mesures** : `SqlAggregation` sum/average/count/min/max +
  `IndicatorDataFormat` Currency/Number… + `Round`) et d'`IndicatorFilter` (la **dimension** de
  regroupement : `EntityPropertyDisplay` + `TreePath` + `FilterType=Aggregate`).
  → `McpCreateIndicator` (1 mesure) / `McpCreateMultiMeasureIndicator` (N mesures) / `McpCreateTypedIndicator`
  (N mesures + type de graphe). **Regroupement par référence** : `TreePath = "EntiteRacine.ProprieteReference"`.
- **Dashboard** = `DynamicDashboard` → `DashCollection` (auto-créée) → `DashboardUser` → `Dashboard` →
  `DashboardReportWidget` (qui porte **un seul** `Indicator`). → `McpCreateDashboard` + `McpAddIndicatorWidget`.
  Un widget « multi-séries » = **un indicateur à plusieurs mesures**, pas plusieurs widgets.

### Lien avec les droits sur les champs (couche FormBuilder ↔ indicateur)
Un indicateur agrège/regroupe des **champs d'entité** : si un champ mesure ou de regroupement n'est pas
exposé/accessible, l'indicateur ne pourra pas le référencer correctement. **Avant** de créer l'indicateur,
s'assurer que les champs concernés existent (`get_entity_fields`) et, au besoin, débloquer leur accès via
les fonctions FormBuilder de ce même skill : `McpSetFieldsApi` (écriture API), `McpGrantTablePermission`
(permission d'entité, sinon 500), `McpSetFieldsUiVisibility`. C'est l'intérêt de co-localiser indicateurs
et config dans **un seul skill** : on peut débloquer un champ puis l'utiliser dans un indicateur sans changer d'outil.

### Recette : indicateur → dashboard → widget
```
0) select_app("demo-vesta-10-5")
1) (si besoin) get_entity_fields(entite) ; McpSetFieldsApi / McpGrantTablePermission sur les champs mesure+dimension
2) McpCreateTypedIndicator(entite, nom, measuresJson, chartType, groupEntite, groupProp, groupTreePath)
3) McpCreateDashboard(moduleId, CODE, "Nom")            // → dynamicDashboardId / dashCollectionId / dashboardId
4) McpAddIndicatorWidget(dashboardId, indicatorId, "Titre", x, y, w, h)
5) McpDiagReportData(indicatorId)                       // vérifier le rendu réel (headers + lignes) sans navigateur
```

### Gotchas catalogués (indicateurs/dashboards — issus de vrais incidents)
- **`Grid` (0) = liste à plat, n'agrège JAMAIS ; `GridMatrice` (16) = tableau agrégé/pivot.**
  `TableOutputGenerationStrategy.cs:21` : `isFlatList = DefaultChartRepresentation == Grid` court-circuite
  l'agrégation (lignes brutes, colonnes = mesures seules). Pour « 1 ligne par dimension avec SUM », utiliser
  **`GridMatrice`** (ou corriger un indicateur existant via `McpSetIndicatorChartType(id, "GridMatrice")`).
- **Nom de widget affiché en GUID** : le libellé doit être écrit avec `resourceKey = "LocalizedName"`
  (chemin de lecture `MapWidgetDtoToViewModel` matche `PropertyName == "LocalizedName"`,
  `DashboardWidget.cs:12`). Un autre `resourceKey` (ex. `"name"`) ⇒ aucun match ⇒ l'UI retombe sur le GUID.
  (`McpAddIndicatorWidget` corrigée le fait déjà.)
- **`IndicatorUserId` = l'`Indicator.Id`** (pas un IndicatorUser) côté `ReportWidgetDataFormModel`.
- **`update_entity_patch` = 500 sur `Indicator` ET `DynamicFunction`** (référence circulaire Mapster, cf. §6).
  Contournement : passer par le service GetSingle+Edit → `McpSetIndicatorChartType` (chart) /
  `McpUpdateFunctionCode` (code fonction).
- **`SqlAggregation` en minuscules** (`count/sum/average/minimum/maximum`), parser avec `Enum.TryParse(.., true)`.
- **`using` inexistant ⇒ échec de compilation à chaud ⇒ retour `null` SILENCIEUX** (aucune entrée AppLog) :
  vérifier que chaque `using` existe. `IServiceManager`/`AppDependencyResolver` sont **auto-injectés** par le wrapper.
- **`McpDiagReportData`** rend le `ReportViewModel` réel d'un indicateur (`GetReportData`,
  `IsPreview=false`) : l'outil de référence pour diagnostiquer un rendu sans ouvrir le navigateur.
- **Config, effet immédiat** : indicateurs/dashboards/widgets/workflow = configuration → **aucun rebuild** (F5 côté UI).
- **⚠️ Éditer un dashboard = réservé au CRÉATEUR.** Tous les endpoints de modification (`EditWidgetsPosition`,
  `Create/Edit/DeleteWidgetOfDashboard`, `Edit/DeleteDashboardInCollection`, `GetDashboardPdfFromId`, +
  `ReportController` partage/suppression) passent par `DashboardService.CanCurrentUserEditDashboard(id)`
  (`DashboardService.cs:1098`) = `Dashboard.CreatedUser.Id == GetCurrent().Id`. Un **non-créateur** (même
  admin / `AccessBuilderRight`, même sur un dashboard **partagé/admin** `IsAdminDashboard=true`) reçoit **HTTP 500
  `{"ErrorMessage":"The user is not authorized"}`** dès qu'il édite (la **consultation** marche). → Débloquer :
  **`McpSetDashboardOwner(dashboardId, userId)`** (transfert de propriété) ou assouplir la règle (code + rebuild).
  ⚠️ Ne pas confondre avec le garde `FromAdminView && !RoleInApp.AccessBuilderRight` (`DashboardController.cs:444/488`)
  qui ne concerne QUE les opérations **niveau collection** (ajouter/réordonner des dashboards dans une collection).

> Détail complet (code de chaque fonction, preuves file:line, enums `ChartRepresentation`/`IndicatorDataFormat`,
> tableau des 20 types de graphes, chaîne validée, récap des IDs) : `references/MCP-Indicators-Dashboards.md`.

## 10. Code métier / business rules sur l'entité — `McpSetEntityCode`

Le code métier serveur d'une entité vit dans les **colonnes d'expression de `DynamicSettings`** (cf.
points d'injection). `McpSetEntityCode(entityName, slot, code)` pose le code sur **n'importe quel slot string**
(par réflexion) : édite `DynamicSettings.<slot>` puis `rm.DynamicSettingsRepository.Edit`.

**Slots C# serveur (12)** : `UsingsExpression`, `Initialize{Create|Edit|Details}Expression`,
`Validation{Create|Edit}Expression`, `Injection{Create|Edit}Expression`, `Post{Create|Edit}Expression`,
`Listener{Create|Edit}Expression`. **Slots front (15)** : `Content{Css|Javascript|Razor}{Create|Edit|Details|List|Global}Expression`.

> ⚠️ **REBUILD OBLIGATOIRE pour les slots C# serveur.** Ces expressions ont un companion `*Compiled`
> + `CodeCompiled` (`DynamicSettings` implémente `IDynamicCodeCompiled`) → elles sont **compilées au build**
> dans l'assembly de l'entité. Après `McpSetEntityCode` sur un slot C#, lancer **`McpBuild`**. (Slots front =
> rendu, F5, pas de rebuild.)

**Contrat des expressions (variables en scope, vérifié sur la démo) :**
- **Validation** (`Validation{Create|Edit}Expression`) : `Model` (`Model.Context.GetFormValue("Champ")`),
  `modelState.AddModelError("Champ", VP.GetResourceDynamic("msg"))` pour **rejeter**. Ex. réel `LegalLeaseAmendment`.
- **Injection / Post** (`Injection*`, `Post*`) : **`entity`** = l'entité en cours (typée) → `entity.Champ = …`,
  `entity.Code = …`. Helpers `VP.*` (`VP.Entities.GetSingleSelect`, `VP.AGL.GetLocalizedTableName`, `VP.SetLogDebug`).
- Helpers transverses : `VP.Entities.*`, `VP.AGL.*`, `VP.GetResourceDynamic`, `VP.Functions.Invoke`.

**✅ Validé en conditions réelles** : `InjectionCreateExpression` = `entity.Reference = (entity.Reference ?? "") + "-INJ";`
sur `McpEvalContrat` → après `McpBuild`, un `create_entity` MCP donne `Reference="CT-INJ-001-INJ"`.
→ **Le chemin REST/MCP `create_entity` DÉCLENCHE bien les business rules de l'entité** (pas seulement le formulaire UI).

### Listeners (`Listener{Create|Edit}Expression`) — pièges validés
- **`DynamicSettings` est PAR VUE.** Une entité a **plusieurs** `DynamicSettings` : la **canonique**
  (`ViewName == null`) + une par vue nommée (ex. `User` : `AllUsers`, `InternalTechnician`, `UserProfile`).
  `McpSetEntityCode` (et tous les helpers) ciblent la **canonique** via
  `GetSingleSelect(x => x.EntityName==… && x.ViewName==null)`. ⚠️ Si **≥2** settings avaient `ViewName==null`,
  ce `GetSingleSelect` **lèverait** « Sequence contains more than one element ».
- **Le listener post-insert/update n'exécute QUE la settings canonique** (`ViewName==null`) :
  `CSharpEngineService.cs:459` filtre `Where(x => x.ViewName == null)`. → Les slots `Listener*Expression`
  des settings **de vue** sont **morts** (jamais exécutés). Corriger un listener = poser le code sur la
  **canonique** (ce que fait `McpSetEntityCode`) **+ `McpBuild`**.
- **Le listener tourne à CHAQUE insert/update de l'entité** (ex. au **login**, la MAJ du `User` le déclenche).
  Pour l'entité `User`, le paramètre typé du code dynamique est **`UserExtension`** (table d'extension
  FormBuilder ; `Persona.User` est d'ailleurs typé `UserExtension`).
- **`VP.GetSingleSelect<TEntity,TResult>(where, select)`** = `SingleOrDefault` → **lève « Sequence contains
  more than one element » sur ≥2 lignes**, `null` sur 0 (`NHRepository.cs:507`). **Fragile** dès que l'unicité
  n'est pas garantie par la donnée (doublons). Variantes « many » :
  - `VP.GetManySelect<TResult,TEntity>(where, select)` — **ordre des génériques INVERSÉ** `<TResult,TEntity>`,
    `[Obsolete]` mais OK, **SANS filtre global** (= même sémantique que `GetSingleSelect`, juste tolérant à 0/N).
  - `VP.Entities.GetManySelect<TEntity,TResult>(select, where, …)` — non-obsolète **mais applique le filtre
    global/périmètre** (peut masquer des lignes) → sémantique différente.
  - Le compilateur dynamique **ne traite PAS les warnings en erreurs** (`CSharpCompilationOptions(…, Release)`
    sans `WithGeneralDiagnosticOption(Error)`, `CSharpEngineService.cs:159`) → un appel `[Obsolete]` **compile**
    (warning CS0618). On peut donc durcir une expression sans changer sa sémantique.
- **✅ Validé** : `User.ListenerEditExpression` faisait `VP.GetSingleSelect<Persona,Persona>(… EntityState.Active …)`
  → crash au login pour tout user à **>1 `Persona` actif**. Durci en `VP.GetManySelect<Persona,Persona>(where, x=>x)`
  + `foreach` (tolère 0/N), puis `McpBuild`. Même en cas d'erreur de compile, le listener est `try/catch`+rollback
  (cf. §1) → **jamais pire que l'état courant**.

### Code de transition workflow — `McpSetWorkflowActionCode`
Le code métier d'une **transition** de workflow vit sur **`EntityWorkflowAction`** dans **deux** slots C#
(⚠️ l'ancienne doc parlait d'un champ `Action` — **inexistant** ; les vrais slots sont) :
- **`ValidationExpression`** : exécutée AVANT la transition (peut bloquer ; contrat type `Model`/`modelState`).
- **`InjectionExpression`** : exécutée à la transition (variable `entity` + helpers `VP.*`).
Chacun a un companion `*Compiled` → **compilé au build** ⇒ **`McpBuild` requis** après pose.
`McpSetWorkflowActionCode(actionCode, slot, code)` résout l'action par son **`Code`**
(`EntityWorkflowActionService.GetMany(x => x.Code == …)`) et écrit le slot (repository).
**✅ Validé** : write set→read→revert sur une action démo (sans build → aucun effet de bord).

**À étendre (chantier suivant)** : statuts de workflow supplémentaires (`EntityWorkflowStatusService.Create`
— contrat documenté §8 d'Indicators) ; conditions d'alerte fines (`EntityPropertyDataAlert` + helper) ;
mapping complet d'une `PropagationRule` (`PersistPropagationMapping`) ; planification d'un batch
(`DynamicBatchService.CreateScheduledTask` → `BatchScheduledTask`) ; visuels Image/File (upload).
**✅ Faits (2026-06)** : `McpAttachFieldPage`, boutons (`McpCreateDynamicButton`/`McpSetButtonCode`/
`McpDeleteDynamicButton`), business rules tous kinds + `McpSetBusinessRuleExtras` + `McpDeleteRuleTree`,
`McpSetTreeLevels`, batchs (`McpCreateBatch`/`McpRunBatch`), **workflow transitions**
(`McpCreateWorkflowAction`/`McpRunWorkflowAction`/`McpDeleteWorkflowAction` ✅ roundtrip),
**visuels** (`McpAddTableWidget`/`McpDeleteTableWidget`), **confidentialité**
(`McpSetTableConfidentiality`/`McpSetRecordConfidentiality`), **PDF** (`McpCreatePdfModel`),
**alertes** (`McpCreateDataAlert`), **aides** (`McpCreateOnlineHelp`).

## 11. Code dynamique PROPRE — pages, code global, framework VP, rendu listes

**Doc complète autoportante : `references/CODE-DynamicPages-VPFramework.md`** (surface d'API VP C# &
VP JS, contrat verbatim de CHAQUE slot d'expression, pipelines de rendu, fonctions validées ✅).
L'essentiel à savoir sans ouvrir la référence :

- **Standard absolu** : tout code dynamique passe par le framework **`VP`** — C# serveur
  (`VP.Entities.GetManySelect<T,R>(…)`, `VP.Mail`, `VP.Files`, `VP.Functions.Invoke`,
  `VP.GetResourceDynamic`, `VP.GetFormattedNumber`…) et JS client
  (`VP.Entities.*`/`VP.Functions.Invoke`/`VP.UI.GetDynamicPage` → enveloppe `{Success, Result}`,
  `appBaseURL`). Jamais de SQL/fetch bruts.
- **Usings auto-injectés** : tous les `VPSoft.*` usuels + Newtonsoft + System.* ; n'ajouter en
  `CodeUsing` que l'exotique (`VPSoft.Domain.Contracts.DynamicPages`, `…Contracts.Rule.Core`…).
  Un using inexistant ⇒ compilation KO ⇒ retour `null` muet.
- **Variables en scope (à ne pas confondre)** : expressions d'entité Injection/Post/Listener →
  **`entity` (typée)** ; Validation → **`modelState` + `Model`** ; Initialize → **`Model`/`this`**
  (form model, extensions `Model.SetDefaultValue/OverrideDropDownList`) ; bouton `DynamicExpression`
  → **`Model` (dynamic)** ; batch → **`Context`** (`IBatchContext`).
- **`Listener{Create|Edit}Expression` = listeners NHibernate post-insert/post-update** (tout canal,
  session dédiée, anti-boucle par Id) — PAS du live formulaire (le live = `ExpressionTriggerFields`).
- **DynamicPage/Widget/PageField** : 1 table (`DynamicPages`), slots `CodeRazor/CodeCss/CodeJavascript`,
  `Code` unique, `RoleInModules` obligatoire pour l'accès URL (sinon `UnauthorizedAccessException`),
  rendu = `<style>` + Razor compilé (`@Model`, `@Html`, `VP.*`) + `<script>` ; `IsIndependent` ⇒ iframe.
  → `McpCreateDynamicPage` + **`McpRenderDynamicPage`** (rendu testable sans navigateur ✅).
  ⚠️ `McpCreateDynamicPage` **ne pose PAS les rôles** → enchaîner **`McpSetDynamicPageRoles(code, "Role1,Role2")`**.
- **Page INTERACTIVE (cockpit CRUD) — patterns validés ✅** (détail : `references/CODE-DynamicPages-VPFramework.md` §11) :
  - **Lecture 100% client OK** : `VP.Entities.GetManySelect/GetSum/GetCount/GetSingleSelect` (retour `{Success,Result}` ;
    select imbriqué `"Envelope.Code"` → clé aplatie **`EnvelopeCode`** ; `where` multi-niveau OK).
  - ⚠️ **`VP.Entities.Create` (JS, endpoint V2) NE LIE PAS LES RÉFÉRENCES** : `VP2Controller.Create` fait
    `Convert.ChangeType(val, propType)` → échoue **silencieusement** sur tout champ référence (code→entité) et souvent
    decimal/enum reçus en JsonElement ; seuls les `string` passent. **Donc inutilisable pour créer une entité liée.**
  - **Créer depuis une page = fonction serveur** appelée via `VP.Functions.Invoke("Fn",[args])`. Helper générique
    **`BdgCreate(entityName, fieldsJson)`** (réflexion + NHibernate : réf chargée par `Code`, enum→`Enum.ToObject(int)`,
    decimal/date castés, `Code` auto, audit). (L'outil MCP `create_entity` = API **v1**, gère les réfs (champ direct=code) si
    `DynamicFieldRole.Api` exposé via `McpSetFieldsApi` — chemin DIFFÉRENT du V2 navigateur.)
  - ⚠️ **Page PLEINE (via menu)** : layout `_Layout-dynamic`→`_Layout-user`, le bundle `VP.js` est chargé **EN BAS** →
    le `<script>` inline de la page tourne AVANT que `VP` existe (`VP is not defined`). → **init dans un poll** :
    `(function boot(){ if(typeof VP==="undefined"||!VP.Entities){setTimeout(boot,50);return;} init(); })()`.
    (Les widgets de carte n'ont pas ce souci : chargés en AJAX après `VP`.)
  - **Menu** : `McpAddPageToMenu(pageCode, moduleId, name, label, icon, order, roleCodesCsv)` (`MenuType.DynamicPage`).
  - **Gros JS/CSS** ⇒ ⭐ **`McpUpdateDynamicPageCode` en TEXTE BRUT, appelée DEPUIS LE NAVIGATEUR** avec le contenu
    de fichiers locaux (`<input type=file>` injecté → `file.text()` → `VP.Functions.Invoke`) : c'est le navigateur qui
    fait l'échappement JSON, donc **aucun base64 nécessaire**. ❌ **NE PAS** faire recopier des chunks base64 par un
    LLM (`McpSetDynamicPageJsB64`) : anti-pattern, ne converge pas (cf. §6 et `references/PORT-DynamicPages-AGLX.md`).
    Toujours `node --check` le JS en local + `McpRenderDynamicPage` après (le rendu inclut le JS verbatim, ne l'exécute pas).
- **Code global / code global module** = `DynamicGlobalCode` (`DynamicModule null` = app ; sinon module).
  Injecté sur TOUTES les pages du layout dynamique : CSS dans le head, Razor avant le body
  (`@Model` inutilisable → utiliser `VP.*`), JS en fin de scripts ; CSS/JS servis en fichiers virtuels
  cachés 1 an (invalidation par `ModifiedDate`). Config pure (F5).
- **Rendu custom des listes (jTable)** : `customDisplay` par colonne (`new Function('data', code)`,
  `data.record` = la ligne), événement **`tableReady`** (CustomEvent après chaque chargement AJAX —
  LE hook à utiliser dans `ContentJavascriptListExpression`), helpers `SetStyles/SetContent/SetScript`.
  Slots de liste : `Global + List` concaténés, injectés avant la table.
- **DynamicButton** : `DynamicExpression` (C# serveur, `Model` = entité de ligne / expando formulaire /
  `{Ids}` en header ; `return false` ⇒ erreur ; objet retourné ⇒ `data.Result` du `JSCallBack`),
  `JSPreCall`/`JSCallBack` (JS client). Positions : Form=0, Section=1, Subsection=2, Table=3, TableHeader=4.
  → `McpCreateDynamicButton` + `McpSetButtonCode` + `McpDeleteDynamicButton` ✅.
- **DynamicBatch** : code wrappé dans `MainVP()` avec **`Context`** (`AddLog/SetReturnMessage/SetProgress/
  CancellationToken`) → `McpCreateBatch` + `McpRunBatch` (exécution + logs sans UI) ✅. Planification via
  `BatchScheduledTask` (écran ScheduledTasks).
- **Page-champ sur un champ** : `McpAttachFieldPage(entity, champ, pageCode, "Add,Edit,Details"|"clear")` ✅.

## 12. Conditionnalités de champs & business rules NO-CODE

**Doc complète autoportante : `references/NOCODE-Conditionality-BusinessRules.md`** (modèle, JSON,
opérateurs, recettes, fonctions ✅ roundtrip validé sur la démo). L'essentiel :

- **Modèle** : `BusinessRule` (sous-classes `HideRule/ShowRule/ReadOnlyRule/MandatoryRule/
  ValidationRule/FilterRule/AffectationRule/PropagationRule/AlertRule` — le ctor pose `Type`) +
  arbres `RuleTree` (JSON `RuleNodeDto` : `kind` group=0/rule=1/expression=2/ruleset=3, `combinator`
  and=0/or=1, `field` `entity.X`/`oldEntity.X`/`parentEntity.X`/`global.X`, `operator`, `values`
  `[{source:0=LITERAL|1=MODEL|2=CONTEXT|3=FUNCTION, value}]`).
- **Une conditionnalité de champ** = BusinessRule `Hide/Show/ReadOnly/Mandatory` avec **`Event=Field(1)`
  et `EntityId = DynamicField.Id`** (Entity=0→DynamicSettings.Id, Section=2→DynamicSection.Id,
  DynamicButton=3→DynamicButton.Id, NavMenu=5, Dashboard=6, EntityWorkflow[Action]=8/9).
- **Recette** : résoudre la cible (`get_many_select("DynamicField","Id","EntityName == \"X\" and
  PropertyName == \"Y\"")`) → `McpCreateBusinessRule(kind, entity, "Field", targetId, nom, treeJson, "")`
  → vérifier `McpGetRuleTree` → F5 (config pure, pas de build). Supprimer : `McpDeleteBusinessRule`
  (cascade arbres ✅). **Compléments par type** : `McpSetBusinessRuleExtras(ruleId, '{"actionTree":…
  /*Filter*/, "affectationSpec":{"source":1,"value":"entity.X"} /*Affectation — ⚠️ spec ParameterValue,
  PAS un arbre*/, "validationMessageKey":"…", "priority":N}')` ✅.
- **Lecture/audit** : les sous-classes sont requêtables par NOM (`get_many_select("HideRule", "Name,
  EntityName,Event,ActivationRuleTreeId", …)`) ; les nœuds d'arbre (`RuleNode`…) NE le sont PAS
  (pas `IEntity`) → `McpGetRuleTree`. `ViewName=null` = règle de la table maître ; sinon par vue.
- **Effet** : Hide/Show/ReadOnly/Mandatory → évalués au formulaire (client) ; Validation → au save ;
  Filter → compilé en prédicat NHibernate (listes + dropdowns de référence). Aucun rebuild.

## 13. Onglets avancés d'une table — VISUELS, confidentialité, PDF, alertes, aides

**Doc complète autoportante : `references/CONFIG-TableExtras.md`** (entités verbatim, fonctions ✅,
pattern préfiltre). Workflow (statuts/transitions/exécution) → `references/MCP-Indicators-Dashboards.md` §8.
L'essentiel :

- **VISUELS (cards au-dessus/en-dessous des listes)** : `DynamicSectionTemplate` (rangée, **par rôle**,
  `IsBeforeTable`) + `DynamicColumnTemplate` (card : Text=1 HTML / **Widget=2** = Id d'un DynamicWidget /
  Image=3 / File=4, `ClassWidth` Bootstrap). Rendu **liste utilisateur** filtré sur les rôles de
  l'utilisateur ; un widget est chargé par `/Users/Page?id=<widgetId>`. → `McpAddTableWidget(entity,
  roleCode, before, pos, '[{"type":"widget","content":"<CodeWidget>"},…]')` ✅ / `McpDeleteTableWidget`.
  ⚠️ **`classWidth` : pour N cards sur UNE ligne, utiliser `"col"`** (le défaut), **pas `col-md-N`** : le
  rendu est `row …gap-2`, donc 4×`col-md-3` (=100%) **+ les gaps débordent → 3+1**. Retrofit d'une section
  existante : `McpSetColumnWidth(sectionId, "col")`. ⚠️ **`RoleInModule` est PAR MODULE** : `McpAddTableWidget`
  doit résoudre le rôle **dans le module de l'entité** (sinon section visible sur la liste user mais
  INTROUVABLE dans l'admin Visuels, qui filtre sur le `RoleInModule.Id` du bon module). Helper corrigé ✅ ;
  retrofit d'une section mal rattachée : `McpSetSectionRole(sectionId, <bon RoleInModule.Id>)`. (détails → CONFIG-TableExtras §1.3).
  **Pattern consultant ⭐ carte KPI cliquable qui préfiltre** : widget Razor (`VP.Entities.GetCount`) + JS au clic.
  🔴 **LE BON PATTERN = persister le filtre en session puis recharger** (PAS un `jtable("load",{jsonFilters})`
  client-only qui ne persiste pas au F5 et n'affiche ni bandeau ni compteur) : `POST /Api/DynamicFilters/SetFiltersSession
  {EntityName, ViewName, JsonFilters}` puis rafraîchir. **Préféré (sans reload, validé)** : `$('.dynamicListContainer').jtable('load')`
  (relit la session) + reconstruire le bandeau de chips comme `saveAndApplyFilter` — 🔴 pour montrer le bandeau faire
  **`bannerEl.classList.remove('d-none')`** (PAS `add('d-flex')` qui collapse la largeur → chip en « +1 »). Variante simple : `location.reload()`
  (le serveur rerend tout depuis la session). « Tous » = `ResetTableFilterSession` (+ `add('d-none')` si sans reload). Format = sortie de
  `serializeFilters` ; **enum multi-select** : `value`/`text`=**`[TechnicalName]`** (ex. `"[Ordered]"`, crochets car
  select multiple, PAS l'int), `operator:"="`, `isKeysValues:"false"`, `valueType:"multi_enum"`, `displayName`.
  ⚠️ `isKeysValues:"true"` = tree-data only (sinon `value=text`→`Enum.Parse` plante). *(Chemin DIRECT `jtable load`→`GetDynamicEntitiesByFilter` :
  l'INT `value:"2"` marche aussi ; serveur lit la clé `type`, pas `valueType`.)* Détails + code widget → CONFIG-TableExtras §1.4.
  (Les anciennes notes `operator:"equal"` étaient FAUSSES.) Carte « Tous » = recharger avec `jsonFilters:"[]"`.
  Constantes : `FilterQuery.cs` (`TYPE_KEY="type"`, `ISKEYSVALUES="isKeysValues"`, `PROPERTY_VALUE_KEY="value"`).
- **FILTRES de liste (bandeau de recherche)** : une colonne devient filtrable ⟺ **`DynamicFieldRole.Filter=true`**
  (par champ/rôle ; `DynamicFieldHelper.cs:608` — PAS automatique depuis l'affichage). → **`McpSetFieldsFilter(entity,
  moduleId, "Champ1,Champ2,…", "true")`** (modèle `McpSetFieldsEditInLine`). Préfiltre permanent d'une vue =
  `DefaultFilters` (string JSON, même forme que ci-dessus) sur la vue. Valeurs d'un enum pour bâtir un filtre =
  `DynamicMultiEnumValues` (`Value` int, `TechnicalName` ; lié au `DynamicMultiEnum` dont `Name`=`<Entité><Prop>`).
- **Confidentialité** : table → `McpSetTableConfidentiality` (`IsConfidentialMaster`/maître) ;
  enregistrement → `McpSetRecordConfidentiality` (`IsConfidential` + `ConfidentialUsers/Roles`).
  Filtre : visible si non-confidentiel OU user/rôle listé (appliqué aussi au MCP, filterQuery `All`).
  ⚠️ toujours s'inclure avant de poser `true`.
- **Workflow transitions** : `McpCreateWorkflowAction(entity, code, nom, fromStatusCode, toStatusCode,
  rolesCsv, couleur, ordre)` → `McpRunWorkflowAction(entity, recordCode, actionCode, commentaire)`
  (statut + `EntityWorkflowHistory` + InjectionExpression) → `McpDeleteWorkflowAction`. ✅ roundtrip
  validé — un workflow Inactive reste exécutable par programme (seul l'affichage est gaté).
- **PDF** : `McpCreatePdfModel(entity, nom, '[{"name":"…","order":1,"content":"<h1>Razor/HTML</h1>"}]',
  "Portrait|Landscape", selected)` → `EntityPDFModel` + `SectionConfigPDF` (rendu EvoPdf). ✅
- **Alertes** : `McpCreateDataAlert(entity, code, nom, "Create|Edit|CreateAndEdit|Delete", sujet,
  message, rolesCsv, active)` — `DataAlert` Notification déclenchée par listener NHibernate
  (Standard/!batch/Active). **Créer Inactive puis activer** ; conditions fines via
  `EntityPropertyDataAlert`/`AlertRule`. ✅
- **Aides** : `McpCreateOnlineHelp(entity, titre, aideListe, aideCreate, aideEdit, aideDetail,
  rolesCsv, vue)` — `HelpOnline`, 4 contextes localisés. ⚠️ 1 seule aide par entité+vue+contexte+rôle
  (visibilité testée par `count == 1`). ✅

## 14. Porter du code de DynamicPage entre instances (10.4 → 10.5, etc.)

**Doc complète : `references/PORT-DynamicPages-AGLX.md`** (validé sur le portage FIB 10.4 → `FIB_INSPECTION` 10.5).

- **Règle d'or : qui transporte les octets ?** Un LLM n'est pas un canal binaire. Le helper serveur n'est jamais
  le goulot — le canal l'est.
- **Push ciblé de quelques pages ⇒ `McpUpdateDynamicPageCode` (texte brut) DEPUIS LE NAVIGATEUR** : fichiers
  locaux → `<input type=file>` injecté → `file.text()` → `VP.Functions.Invoke`. Puis `McpRenderDynamicPage` + test visuel.
- **Portage d'un module entier / mise en prod ⇒ route native AGLX** : `sm.AglxMainService.Export` / `ExportAsFile`
  (.aglx = ZIP) + preview du diff + `Import` avec **backup et logs**. L'AGLX porte le code **ET** les rôles
  (`RoleInModuleAglxIds`), le dossier et les libellés localisés. ⚠️ export **par module** (8,4 Mo pour Portfolio)
  mais **import sélectif jusqu'à la propriété** (`ModuleTargets`→`SelectedTargets`→`SelectedProperties`) ;
  `ModuleIds` = **AglxId** ≠ Id d'entité ; **aucun contrôle de `SourceVersion`** (10.4→10.5 ni bloqué ni protégé) ;
  `Import` exige un `IFormFile` ⇒ opération UI, pas pur MCP. Sonde : `McpAglxExportProbe(mode, moduleAglxIds)`.
- **Après tout portage** : vérifier le **boot DOM** des pages pleines et réaligner les `grid-template-columns`
  si un enfant de grille a été supprimé.
