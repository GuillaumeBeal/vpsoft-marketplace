# FormBuilder via MCP — Workflow dynamique & DynamicFunctions minimales

Ce document décrit le workflow de création de tables / champs / build dans VPSoft 10.5,
puis fournit le code C# de **3 DynamicFunctions minimales** permettant de piloter ce workflow
depuis un MCP (`invoke_vpsoft_function`).

---

## 1. Vue d'ensemble du workflow dynamique

La création d'une structure dynamique se fait en **3 phases distinctes** :

| Phase | Endpoint Web | Effet réel |
|-------|--------------|------------|
| 1. Créer une table | `POST /api/FormBuilder/AddDynamicEntity` | Écrit un `ConfigurationDescriptor` + un `DynamicSettings` (config uniquement) |
| 2. Créer un champ | `POST /api/FormBuilder/CreateDynamicField` | Écrit un `DynamicField` + propriété dans le descriptor (config uniquement) |
| 3. Build | `POST /api/FormBuilder/BuildConfiguration?processToken=...` | Génère les DLL d'entités, **migre le schéma SQL**, recompile le code |

> **Important :** les phases 1 et 2 n'écrivent que de la **configuration**.
> Tant que le **Build** (phase 3) n'a pas été exécuté avec succès, la table/les colonnes
> n'existent pas physiquement en base et les entités ne sont pas utilisables.

> **⚠️ Phase 4 — droits (obligatoire pour l'API REST de données).** Après le build, la table
> existe en base mais n'est **pas** accessible via l'API REST dynamique (`/api/v1/...`,
> utilisée par `create_entity` / `get_many_select` du MCP). Contrairement à l'UI
> (`AddDynamicEntity`), les fonctions FormBuilder via MCP **ne créent aucun droit**. Il faut
> donc, après le `McpBuild`, octroyer **deux couches de permissions** (cf. §3.8) :
> 1. **`RolePermission`** (niveau entité) — sans elle, l'API renvoie **HTTP 500**.
> 2. **`DynamicFieldRole.Api = true`** (niveau champ) — sans elle, les champs sont
>    **silencieusement ignorés** en lecture/écriture (valeurs `null`, `Code` auto-GUID).

### Routage

`ApiController` porte `[Route("api/[controller]/[action]")]`, donc
`FormBuilderController.AddDynamicEntity` ⇒ `/api/FormBuilder/AddDynamicEntity`.
Le contrôleur est protégé par `[Authorize(Roles = ADMIN,USER)]` + `[AuthorizeToConfigure(canAccessBuilder:true)]`.

### Suivi de progression du Build

- `BuildConfiguration` crée un `HotReloadStepByStepProcess(processToken, …)` puis appelle
  `FormBuilderService.BuildVersion(processToken, process)` **en fire-and-forget** (non `await`).
- La progression est poussée par **SignalR** sur le hub `ProgressHub`, mappé à
  `/api/ProgressionHub` (`VPSoftSettings.SignalR.ProgressionHubName = "api/ProgressionHub"`).
  Le client rejoint le groupe via `JoinProgressGroup(processToken)`.
- Étapes (`HotReloadStepState`) : `Init=0`, `Validation=15`, `EntityDllGeneration=25`,
  `MigrateModel=50`, `CodeDllGeneration=75`, `Finalizing=90`, `Error=100`.
- `ProcessUserService` (cache mémoire par utilisateur) permet aussi de lire l'état via
  `GetState(token)` / `GetProgress(token)`.

> **Pour un MCP** (pas de client SignalR) : le plus simple est de rendre le build
> **synchrone** dans la DynamicFunction en faisant `await BuildVersion(...)`.
> `BuildVersion` exécute tout son travail dans un `await Task.Run(...)` avec un scope DI dédié,
> donc l'attendre est sûr et renvoie la main une fois le build terminé.

---

## 2. Mécanique d'exécution d'une DynamicFunction

Chemin d'appel MCP :

```
invoke_vpsoft_function(name, params)
  → POST /Api/V2/VP/Functions/Invoke           (VP2Controller, [ApiAuthorizeByToken])
    → VP.Functions.InvokeAsync(name, params)   (VPSoft.Utils.Extensions.VP)
      → DynamicFunctionService.EvalDynamicFunction(Async) / RunDynamicFunctionAsync
        → DLL mergée si dispo, sinon compilation à chaud (CSharpEngineService)
```

### Génération du code (CSharpEngineService + DynamicCodeBuilderHelper)

Le `CodeFunction` saisi est **enveloppé** par `GenerateStandaloneDynamicFunctionCode` :

```
<Usings par défaut>
<CodeUsing supplémentaire>
namespace <FormBuilderDynamicFunctionsNameSpace>
{
    <CodeClass>
    public static class <prefix><Name>
    {
        public static <ReturnValue|void> <Name>(<Parameters>)
        {
            try { <CodeFunction> }
            catch (Exception ex) { VP.GetLogExceptionInfo(...); return default(<ReturnValue>); }
        }
    }
}
```

Points clés à respecter dans le `CodeFunction` :

- **Synchrone non-void** : signature = `public static <ReturnValue> <Name>(...)` ⇒ il faut `return <ReturnValue>;`.
- **Async non-void** : la signature est **forcée** à `public static async Task<System.Object> <Name>(...)` ⇒ il faut `return <object>;`.
- **Void** : pas de `return` de valeur.
- L'accès aux services se fait via :
  `var sm = AppDependencyResolver.GetService<IServiceManager>();`

### Usings déjà fournis par défaut (extrait utile)

`System`, `System.Linq`, `System.Collections.Generic`, `System.Threading.Tasks`,
`Newtonsoft.Json`, `VPSoft.Domain.Utils.Dependency` (→ `AppDependencyResolver`),
`VPSoft.Services.Abstractions` (→ `IServiceManager`), `VPSoft.Utils.Extensions` (→ `VP`),
`VPSoft.Domain.Models.Builder`, `VPSoft.Domain.Helpers`, `VPSoft.Domain.Enums.Builder`.

### Usings à AJOUTER explicitement (non présents par défaut)

| Type utilisé | Namespace à mettre dans `CodeUsing` |
|--------------|--------------------------------------|
| `DynamicSettingsCreateFormModel` | `VPSoft.Domain.Contracts.DynamicModules` |
| `FormPropertyModel`, `DynamicFieldConfigFormModel` | `VPSoft.Domain.Contracts.DynamicFields` |
| `FormProperty`, `ReferenceRelation` | `VPSoft.Domain.Utils.Builder.Descriptor` |
| `FormPropertyType`, `FormPropertyCategory` | `VPSoft.Domain.Enums` |
| `ReflectionHelper` | `VPSoft.Domain.Helpers` |
| `GetEntityTypeFormBuilderWhenIsManaged` (extension) | `VPSoft.Domain.Helpers.Extensions` |

### ⚠️ Règle MCP : tous les paramètres doivent être `string`

L'invocation MCP (`POST /Api/V2/VP/Functions/Invoke`) reçoit les paramètres dans un
`List<object>` (`DynamicFunctionJson.CurrentParameters`) et le client MCP les envoie **tous
en chaînes**. Le binding final passe par `Type.InvokeMember` avec le binder par défaut
(`CSharpEngineService.cs:332`), qui **ne convertit pas** `string → bool` ni `string → int`
(erreur runtime : *"Object of type 'System.String' cannot be converted to type
'System.Boolean'"*).

**Conséquence** : pour toute DynamicFunction destinée au MCP, déclarer **chaque paramètre en
`string`** et convertir dans le corps :

```csharp
bool b   = (s == "true" || s == "1" || s == "True");   // string -> bool
int  n   = int.Parse(s);                               // string -> int
Guid g   = Guid.Parse(s);                              // string -> Guid
```

> Validé en conditions réelles : une signature avec `bool isAsync` renvoie une erreur de
> binding ; la même en `string isAsync` (parsé dans le corps) fonctionne.
> Les signatures de §3 ci-dessous appliquent déjà cette règle.

---

## 3. Les DynamicFunctions minimales

> Convention de saisie dans l'écran DynamicFunction :
> **Name** = nom de la fonction, **Parameters** = signature C# des paramètres,
> **ReturnValue** = type de retour (`System.Object` recommandé pour MCP),
> **CodeUsing** = usings supplémentaires, **CodeFunction** = corps,
> **IsAsync** = coché uniquement pour `McpBuild`.

---

### 3.0 `McpCreateFunction` — fonction *bootstrap* (à installer une seule fois)

Fonction « méta » : elle crée d'**autres** DynamicFunctions. On l'installe **manuellement
une seule fois** (écran DynamicFunction de l'application), puis on l'appelle via le MCP
(`invoke_vpsoft_function("McpCreateFunction", [...])`) pour déployer toutes les autres
(`McpCreateTable`, `McpCreateField`, `McpBuild`, …) sans repasser par l'UI.

S'appuie sur `DynamicFunctionService.Create(DynamicFunction)` (`DynamicFunctionService.cs:44`,
simple `Save`). Le corps (`CodeFunction`) est compilé **à chaud au premier appel**
(`EvalDynamicFunction`), donc aucune étape de *merge* n'est requise pour rendre la fonction
invocable.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateFunction` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string name, string parameters, string returnValue, string codeUsing, string codeFunction, string isAsync, string moduleId, string folderId` |
| **CodeUsing** | `using VPSoft.Domain.Enums;`  *(pour `EntityState` ; `DynamicFunction` et `DynamicFolder` sont déjà dans les usings par défaut)* |

> `folderId` **ou** `moduleId` est requis (cible où ranger la fonction).
> Si `folderId` est vide, on prend le dossier racine du module `moduleId`.

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

if (string.IsNullOrWhiteSpace(name))
    return new { success = false, message = "name requis" };

// Unicite du nom (meme controle que DynamicFunctionController.Create)
if (sm.DynamicFunctionService.GetCountByFilter(x => x.Name == name) > 0)
    return new { success = false, message = "Une fonction nommee '" + name + "' existe deja." };

// Resolution du dossier cible
DynamicFolder folder = null;
if (!string.IsNullOrEmpty(folderId))
    folder = sm.DynamicFolderService.GetSingle(x => x.Id == Guid.Parse(folderId));
else if (!string.IsNullOrEmpty(moduleId))
    folder = sm.DynamicFolderService.GetSingle(x => x.ParentFolder == null && x.DynamicModule.Id == Guid.Parse(moduleId));

if (folder == null)
    return new { success = false, message = "Dossier cible introuvable (fournir folderId ou moduleId valide)." };

var fn = new DynamicFunction();
fn.Name = name;
fn.Parameters = parameters;                                   // ex: "string a, int b"  (vide si aucun)
fn.ReturnValue = string.IsNullOrEmpty(returnValue) ? "System.Object" : returnValue;
fn.CodeUsing = codeUsing;                                     // usings supplementaires (multi-lignes ok)
fn.CodeFunction = codeFunction;                               // corps de la fonction
fn.IsAsync = (isAsync == "true" || isAsync == "1" || isAsync == "True");
fn.EntityState = EntityState.Active;
fn.Code = Guid.NewGuid().ToString();                          // code entite (identifiant)
fn.DynamicFolder = folder;

bool ok = sm.DynamicFunctionService.Create(fn);

return new
{
    success = ok,
    functionId = ok ? (Guid?)fn.Id : null,
    name = fn.Name,
    folderId = folder.Id,
    message = ok
        ? "DynamicFunction creee. Invocable via le MCP (compilation a chaud au 1er appel)."
        : "Echec de creation."
};
```

**Notes :**
- C'est la **seule** fonction qu'il faut saisir à la main. Toutes les autres peuvent ensuite
  être créées par appel MCP à `McpCreateFunction`.
- `codeUsing` / `codeFunction` se passent en chaînes (les sauts de ligne sont autorisés). Pour
  une fonction `IsAsync`, passer `isAsync = "true"` et utiliser `await` dans `codeFunction`.
- La compilation réelle a lieu à l'invocation : si `codeFunction` ne compile pas, c'est au
  **premier appel** de la fonction créée que l'erreur remonte (pas à la création).
- Alternative sans aucune saisie UI : créer directement l'entité `DynamicFunction` via le MCP
  `create_entity("DynamicFunction", { Name, Parameters, ReturnValue, CodeUsing, CodeFunction,
  IsAsync, EntityState, DynamicFolder_Code_ })`. `McpCreateFunction` reste utile pour scripter
  la création côté serveur avec le contrôle d'unicité.

---

### 3.1 `McpCreateTable` — créer une table

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateTable` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string className, string entityClassType, string isTree, string withName, string moduleId, string folderId` |
| **CodeUsing** | `using VPSoft.Domain.Contracts.DynamicModules;` |

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

Guid? moduleGuid = string.IsNullOrWhiteSpace(moduleId) ? (Guid?)null : Guid.Parse(moduleId);
Guid? folderGuid = string.IsNullOrWhiteSpace(folderId) ? (Guid?)null : Guid.Parse(folderId);

bool isTreeBool   = (isTree == "true" || isTree == "1" || isTree == "True");
bool withNameBool = (withName == "true" || withName == "1" || withName == "True");

// getDefaultModuleFolderIfNull=true, withModule=true
var folder = sm.DynamicFolderService.GetDynamicFolder(folderGuid, moduleGuid, true, true);
if (folder == null)
    return new { success = false, message = "DynamicFolder/Module introuvable" };

if (!sm.DynamicSettingsService.IsEntityNameValid(className))
    return new { success = false, message = "Nom de classe invalide ou deja existant : " + className };

var model = new DynamicSettingsCreateFormModel
{
    ClassName = className,
    // "Entity" ou "EntityAudit" (audit = trace des modifications). Defaut: EntityAudit
    SelectedEntityClassType = string.IsNullOrWhiteSpace(entityClassType) ? "EntityAudit" : entityClassType,
    IsTree = isTreeBool,
    WithName = withNameBool,
    DynamicModuleId = moduleGuid,
    DynamicFolderId = folderGuid
};

Guid? newId = sm.DynamicSettingsService.CreateDynamicSettingsCustom(model, folder);

return new
{
    success = newId != null,
    dynamicSettingsId = newId,
    className = className,
    message = newId != null ? "Table creee (config). Lancer McpBuild pour materialiser." : "Echec creation"
};
```

**Notes :**
- `CreateDynamicSettingsCustom` force `WithCode = true` (un champ `Code` est toujours créé,
  servant de `IsEntityCode` + `IsEntityName`). Si `IsTree = true`, `WithName` est forcé à `true`.
- `SelectedEntityClassType` accepte `"Entity"` ou `"EntityAudit"` (enum `EntityClassType`).
- Le résultat n'est **pas** encore en base : il faut appeler `McpBuild`.
- ⚠️ **`isTree=true` ne crée PAS les niveaux de l'arbre** → appeler **`McpSetTreeLevels`** (§3.1bis),
  sinon l'arbre est inutilisable côté UI (aucun niveau défini).

---

### 3.1bis `McpSetTreeLevels` — créer les NIVEAUX d'une table arbre ✅

Une table arbre a besoin d'une config **`TreeConf`** (1 par `EntityName` :
`VPSoft.Domain/Models/Entities/Trees/TreeConf.cs` — `TreeConfType` WithPerimeter=0/FullAccess=1,
`CodeResearch`) et de **`TreeConfLevel`** (1 ligne PAR NIVEAU : `TreeLevel` 1-based, `Color`,
nom localisé via `CultureParameters`). C'est ce que fait l'écran de config arbre
(`TreeConfigController.CreateLevel`, `VPSoft.Presentation/Controllers/TreeConfigController.cs:116-156`).
**Maximum 10 niveaux** (`Consts.Tree.MAX_TREE_LEVEL`, `Consts.cs:427`). La création de la table
(`McpCreateTable isTree`) n'en crée AUCUN.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetTreeLevels` |
| **Parameters** | `string entityName, string levelsJson, string treeConfType, string codeResearch` |
| **CodeUsing** | *(aucun — tout est dans les usings par défaut)* |

- `levelsJson` : **soit un nombre** (`"3"` → « Niveau 1..3 », couleurs blanches), **soit un tableau JSON**
  `[{"name":"Pays","color":"#1f77b4"},{"name":"Région"},…]` (couleur défaut `#ffffff`).
- `treeConfType` : `"FullAccess"` (défaut) ou `"WithPerimeter"` ; `codeResearch` : `"true"|"false"`.
- **Idempotent** : crée le `TreeConf` s'il manque, AJOUTE uniquement les `TreeLevel` manquants
  (jamais de suppression) — relancer avec un nombre plus grand AJOUTE les niveaux suivants.

**CodeFunction :**

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    if (string.IsNullOrWhiteSpace(entityName)) return "ERR entityName requis";
    var names = new List<string>(); var colors = new List<string>();
    string lj = (levelsJson ?? "").Trim(); int count;
    if (int.TryParse(lj, out count)) {
        for (int i = 1; i <= count; i++) { names.Add("Niveau " + i); colors.Add("#ffffff"); }
    } else {
        var arr = JArray.Parse(lj);
        foreach (var it in arr) { names.Add((string)(it["name"] ?? ("Niveau " + (names.Count + 1)))); colors.Add((string)(it["color"] ?? "#ffffff")); }
    }
    if (names.Count == 0) return "ERR aucun niveau demande";
    if (names.Count > 10) return "ERR max 10 niveaux (Consts.Tree.MAX_TREE_LEVEL)";
    var treeConf = rm.TreeConfRepository.GetSingle(x => x.EntityName == entityName, x => x.TreeConfLevels);
    if (treeConf == null) {
        treeConf = new TreeConf { EntityName = entityName,
            TreeConfType = (treeConfType == "WithPerimeter") ? TreeConfType.WithPerimeter : TreeConfType.FullAccess,
            CodeResearch = codeResearch == "true" };
        rm.TreeConfRepository.Save(treeConf);
        treeConf = rm.TreeConfRepository.GetSingle(x => x.EntityName == entityName, x => x.TreeConfLevels);
    }
    var added = new List<int>(); var skipped = new List<int>();
    for (int i = 1; i <= names.Count; i++) {
        if (treeConf.TreeConfLevels.Any(x => x.TreeLevel == i)) { skipped.Add(i); continue; }
        var level = new TreeConfLevel { TreeLevel = i, TreeConf = treeConf, Color = colors[i - 1] };
        var rc = new ResourceCultureJson { resourceKey = "LocalizedName", resourceValues = new List<ResourceCultureValueJson> {
            new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = names[i - 1] },
            new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = names[i - 1] } } };
        sm.CultureParameterService.SaveOrUpdate(rc, level.Id, typeof(TreeConfLevel).GetProperty("LocalizedName"));
        treeConf.TreeConfLevels.Add(level);
        added.Add(i);
    }
    rm.TreeConfRepository.Edit(treeConf);
    return JsonConvert.SerializeObject(new { success = true, treeConfId = treeConf.Id, entityName = entityName, added = added, skipped = skipped, totalLevels = treeConf.TreeConfLevels.Count });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**✅ Test validé (démo, table `McpDemoArbre`)** :
`McpCreateTable("McpDemoArbre","EntityAudit","true","true","<moduleId>","")` →
`McpSetTreeLevels("McpDemoArbre","[{\"name\":\"Pays\",\"color\":\"#1f77b4\"},{\"name\":\"Région\",\"color\":\"#ff7f0e\"},{\"name\":\"Site\",\"color\":\"#2ca02c\"}]","FullAccess","false")`
→ `added:[1,2,3]` ; ré-appel `McpSetTreeLevels("McpDemoArbre","5",…)` → `added:[4,5], skipped:[1,2,3]`
(idempotence + forme « nombre ») ; vérif `get_many_select("TreeConfLevel","TreeLevel,Color,LocalizedName",
"TreeConf.EntityName == \"McpDemoArbre\"")` → 5 lignes nommées ; `McpBuild` → la table expose
`Level`, `ParentId`, `ParentCode`, `TreeBranchPathCode`, `TreeDataExtension` (classe `McpDemoArbreTreeData`
générée). **Séquence recommandée : `McpCreateTable(isTree)` → `McpSetTreeLevels` → `McpBuild`.**
(`TreeConf` est de la config par `EntityName`, posable avant ou après build ; F5 côté UI.)

---

### 3.2 `McpCreateField` — créer un champ

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateField` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string dynamicSettingsId, string fieldName, string fieldType, string nullable, string isUnique, string entityNameSelected, string referenceRelationSelected` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums;
using VPSoft.Domain.Contracts.DynamicFields;
using VPSoft.Domain.Utils.Builder.Descriptor;
using VPSoft.Domain.Helpers;
using VPSoft.Domain.Helpers.Extensions;
```

**Valeurs de `fieldType` (`FormPropertyType`) :**
`String=0`, `Integer=1`, `DateTime=2`, `Boolean=3`, `Decimal=4`, `DynamicEnum=5`,
`Enum=6`, `Entity=7`, `MultiCulture=8`, `Date=12`, `File=13`.
Pour `Entity=7` : renseigner `entityNameSelected` (nom de l'entité cible) et
`referenceRelationSelected` (`ReferenceRelation` : `Reference=0`, etc.).

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

Guid dsGuid = Guid.Parse(dynamicSettingsId);
string tableName = sm.DynamicSettingsService.GetSingleSelect(x => x.Id == dsGuid, x => x.EntityName);
if (string.IsNullOrEmpty(tableName))
    return new { success = false, message = "DynamicSettings introuvable : " + dynamicSettingsId };

int fieldTypeInt = int.Parse(fieldType);
int referenceRelationSelectedInt = string.IsNullOrWhiteSpace(referenceRelationSelected) ? 0 : int.Parse(referenceRelationSelected);
bool nullableBool = (nullable == "true" || nullable == "1" || nullable == "True");
bool isUniqueBool = (isUnique == "true" || isUnique == "1" || isUnique == "True");

var fp = new FormProperty();
fp.Name = fieldName;
fp.Category = FormPropertyCategory.Standard;
fp.Status = FormPropertyStatus.ADDED;
fp.Type = (FormPropertyType)fieldTypeInt;

Type entityTypeFormBuilder = ReflectionHelper.GetTypeEntity(tableName).GetEntityTypeFormBuilderWhenIsManaged();
fp.TableName = sm.SessionFactoryService.GetTableName(entityTypeFormBuilder);

fp.IsUnique = isUniqueBool;
fp.Nullable = nullableBool;
fp.EntityNameSelected = entityNameSelected;
fp.ReferenceRelationSelected = referenceRelationSelectedInt;

// Regles metier reprises du FormBuilderController.CreateDynamicField
if (fp.Type == FormPropertyType.Boolean)
{
    fp.Nullable = false;
    fp.HasDefaultValue = true;
}
if (fp.Type == FormPropertyType.Date || fp.Type == FormPropertyType.DateTime || fp.Type == FormPropertyType.Boolean)
    fp.IsUnique = false;
if (fp.IsUnique || fp.Type == FormPropertyType.Date || fp.Type == FormPropertyType.DateTime)
    fp.Nullable = true;

if (fp.Type == FormPropertyType.Entity)
{
    Type entityTypeSelected = ReflectionHelper.GetTypeEntity(fp.EntityNameSelected);
    fp.EntityFullName = entityTypeSelected.FullName;
    fp.ReferenceRelation = (ReferenceRelation)referenceRelationSelectedInt;
    fp.EntityTableName = sm.SessionFactoryService.GetTableName(entityTypeSelected);
    fp.Nullable = true;
}

string errorValidationMessage;
if (!fp.ValidateConstraintForCreate(tableName, out errorValidationMessage))
    return new { success = false, message = errorValidationMessage };

var fcd = sm.ConfigurationDescriptorService
    .GetOrCreateInheritedConfigurationDescriptorFromType(ReflectionHelper.GetTypeEntity(tableName));

if (sm.DynamicFieldService.IsPropertyNameExists(fcd, fp, tableName))
    return new { success = false, message = "Le champ existe deja : " + tableName + "." + fieldName };

var model = new FormPropertyModel
{
    DynamicSettingsId = dsGuid,
    TableName = tableName,
    Property = fp,
    DynamicFieldConfigFormModel = new DynamicFieldConfigFormModel()
};

var newField = sm.ConfigurationDescriptorService.AddPropertyWorkflow(model, fcd, fp, out _);

return new
{
    success = newField != null,
    fieldId = newField != null ? (Guid?)newField.Id : null,
    table = tableName,
    fieldName = fieldName,
    message = newField != null ? "Champ cree (config). Lancer McpBuild." : "Echec creation champ"
};
```

**Notes :**
- On part de `dynamicSettingsId` (identique à l'URL d'origine `CreateDynamicField?dynamicSettingsId=...`)
  et on en déduit `tableName`, exactement comme le contrôleur.
- Pour les champs **Enum** voir `McpCreateEnumField` (§3.3), pour les champs
  **Expression** voir `McpCreateExpressionField` (§3.4).
- Pour un champ **Entity avec sa référence inverse** (collection créée automatiquement côté
  entité cible), voir `McpCreateReferenceField` (§3.6).

---

### 3.3 `McpCreateEnumField` — créer un champ Enum (liste de valeurs)

Crée un champ de type `DynamicMultiEnum` (`FormPropertyType = 11`) avec sa liste de valeurs.
Réplique le bloc `DynamicMultiEnum` du `FormBuilderController.CreateDynamicField` (l.1441-1471).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateEnumField` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string dynamicSettingsId, string fieldName, string isMultiSelect, string isShared, string enumName, string enumValuesJson` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums;
using VPSoft.Domain.Contracts.DynamicFields;
using VPSoft.Domain.Contracts.DynamicEnums;
using VPSoft.Domain.Utils.Builder.Descriptor;
using VPSoft.Domain.Helpers;
using VPSoft.Domain.Helpers.Extensions;
```

**Format de `enumValuesJson`** (tableau JSON, désérialisé via `Newtonsoft.Json`) :

```json
[
  { "TechnicalName": "Rouge", "Value": 1, "LocalizedName": "Rouge", "IsDefault": true },
  { "TechnicalName": "Vert",  "Value": 2, "LocalizedName": "Vert",  "IsDefault": false }
]
```

> `Value` doit être une puissance de 2 si `isMultiSelect = true` (combinable en flags).
> `TechnicalName` doit être un nom C# valide et différent de `none`.

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

// Params MCP => toujours en string, on parse ici
bool isMultiSelectBool = (isMultiSelect == "true" || isMultiSelect == "1" || isMultiSelect == "True");
bool isSharedBool = (isShared == "true" || isShared == "1" || isShared == "True");

Guid dsGuid = Guid.Parse(dynamicSettingsId);
string tableName = sm.DynamicSettingsService.GetSingleSelect(x => x.Id == dsGuid, x => x.EntityName);
if (string.IsNullOrEmpty(tableName))
    return new { success = false, message = "DynamicSettings introuvable : " + dynamicSettingsId };

var values = JsonConvert.DeserializeObject<List<DynamicMultiEnumValuesFormModel>>(enumValuesJson);
if (values == null || values.Count == 0)
    return new { success = false, message = "enumValuesJson vide ou invalide" };

// Validation des TechnicalName (unicité + nom C# valide, != none)
var technicalNames = values.Select(v => v.TechnicalName).Distinct().ToList();
if (technicalNames.Count != values.Count)
    return new { success = false, message = "TechnicalName en double" };
foreach (var tn in technicalNames)
{
    if (string.Equals(tn, "none", StringComparison.OrdinalIgnoreCase) || !StringMatchHelper.IsValidCsharpName(tn))
        return new { success = false, message = "TechnicalName invalide : " + tn };
}

var dynamicMultiEnum = new DynamicMultiEnumFormModel
{
    Name = string.IsNullOrWhiteSpace(enumName) ? fieldName : enumName,
    IsShared = isSharedBool,
    IsMultiSelect = isMultiSelectBool,
    DynamicMultiEnumValuesFormModelList = values
};

// Unicité d'un enum partagé portant le meme nom
if (dynamicMultiEnum.IsShared)
{
    var sameShared = sm.DynamicMultiEnumService.GetCountByFilter(x =>
        x.IsShared && x.Name == dynamicMultiEnum.Name && x.Id != dynamicMultiEnum.Id);
    if (sameShared > 0)
        return new { success = false, message = "Un enum partagé du meme nom existe déjà" };
}

var fp = new FormProperty();
fp.Name = fieldName;
fp.Category = FormPropertyCategory.Standard;
fp.Status = FormPropertyStatus.ADDED;
fp.Type = FormPropertyType.DynamicMultiEnum;

Type entityTypeFormBuilder = ReflectionHelper.GetTypeEntity(tableName).GetEntityTypeFormBuilderWhenIsManaged();
fp.TableName = sm.SessionFactoryService.GetTableName(entityTypeFormBuilder);

// Regles imposées par le contrôleur pour un enum
fp.Nullable = false;
fp.HasDefaultValue = true;
fp.DefaultValue = values.Where(v => v.IsDefault).Sum(v => v.Value).ToString();
fp.DynamicEnumIsMultiSelect = dynamicMultiEnum.IsMultiSelect;

string errorValidationMessage;
if (!fp.ValidateConstraintForCreate(tableName, out errorValidationMessage))
    return new { success = false, message = errorValidationMessage };

var fcd = sm.ConfigurationDescriptorService
    .GetOrCreateInheritedConfigurationDescriptorFromType(ReflectionHelper.GetTypeEntity(tableName));

if (sm.DynamicFieldService.IsPropertyNameExists(fcd, fp, tableName))
    return new { success = false, message = "Le champ existe déjà : " + tableName + "." + fieldName };

// Persiste l'enum puis le rattache à la propriété
var savedEnum = sm.DynamicMultiEnumService.SaveOrUpdateDynamicMultiEnum(dynamicMultiEnum);
fp.DynamicMultiEnum = savedEnum;

var model = new FormPropertyModel
{
    DynamicSettingsId = dsGuid,
    TableName = tableName,
    Property = fp,
    DynamicMultiEnum = dynamicMultiEnum,
    DynamicFieldConfigFormModel = new DynamicFieldConfigFormModel()
};

var newField = sm.ConfigurationDescriptorService.AddPropertyWorkflow(model, fcd, fp, out _);

return new
{
    success = newField != null,
    fieldId = newField != null ? (Guid?)newField.Id : null,
    table = tableName,
    fieldName = fieldName,
    values = values.Count,
    message = newField != null ? "Champ enum créé (config). Lancer McpBuild." : "Echec création champ enum"
};
```

---

### 3.4 `McpCreateExpressionField` — créer un champ Expression (calculé)

Crée un champ calculé (`Category = ExpressionField`) basé sur une lambda C#.
Réplique le bloc `ExpressionField` du contrôleur (l.1370-1378).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateExpressionField` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string dynamicSettingsId, string fieldName, string returnType, string referenceLambda, string entityReturnExpressionLambda` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums;
using VPSoft.Domain.Contracts.DynamicFields;
using VPSoft.Domain.Utils.Builder.Descriptor;
using VPSoft.Domain.Helpers;
using VPSoft.Domain.Helpers.Extensions;
```

- `returnType` = `FormPropertyType` du résultat de l'expression (`String=0`, `Integer=1`, `Decimal=4`, …).
- `referenceLambda` = la lambda d'expression (ex : `x => x.Quantite * x.PrixUnitaire`).
- `entityReturnExpressionLambda` = entité de retour ; si vide, le contrôleur utilise `"AppSettings"`.

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

// Params MCP => toujours en string, on parse ici
int returnTypeInt = int.Parse(returnType);

Guid dsGuid = Guid.Parse(dynamicSettingsId);
string tableName = sm.DynamicSettingsService.GetSingleSelect(x => x.Id == dsGuid, x => x.EntityName);
if (string.IsNullOrEmpty(tableName))
    return new { success = false, message = "DynamicSettings introuvable : " + dynamicSettingsId };

var fp = new FormProperty();
fp.Name = fieldName;
fp.Category = FormPropertyCategory.ExpressionField;
fp.Status = FormPropertyStatus.ADDED;
fp.Type = (FormPropertyType)returnTypeInt;

Type entityTypeFormBuilder = ReflectionHelper.GetTypeEntity(tableName).GetEntityTypeFormBuilderWhenIsManaged();
fp.TableName = sm.SessionFactoryService.GetTableName(entityTypeFormBuilder);

// Reglage imposé par le contrôleur pour un ExpressionField
fp.EntityNameSelected = string.IsNullOrWhiteSpace(entityReturnExpressionLambda) ? "AppSettings" : entityReturnExpressionLambda;
fp.ReferenceLambda = referenceLambda;
fp.Nullable = true;
fp.ReferenceRelation = ReferenceRelation.Reference;
fp.ReferenceRelationSelected = 0;

string errorValidationMessage;
if (!fp.ValidateConstraintForCreate(tableName, out errorValidationMessage))
    return new { success = false, message = errorValidationMessage };

var fcd = sm.ConfigurationDescriptorService
    .GetOrCreateInheritedConfigurationDescriptorFromType(ReflectionHelper.GetTypeEntity(tableName));

if (sm.DynamicFieldService.IsPropertyNameExists(fcd, fp, tableName))
    return new { success = false, message = "Le champ existe déjà : " + tableName + "." + fieldName };

var model = new FormPropertyModel
{
    DynamicSettingsId = dsGuid,
    TableName = tableName,
    Property = fp,
    DynamicFieldConfigFormModel = new DynamicFieldConfigFormModel()
};

var newField = sm.ConfigurationDescriptorService.AddPropertyWorkflow(model, fcd, fp, out _);

return new
{
    success = newField != null,
    fieldId = newField != null ? (Guid?)newField.Id : null,
    table = tableName,
    fieldName = fieldName,
    message = newField != null ? "Champ expression créé (config). Lancer McpBuild." : "Echec création champ expression"
};
```

---

### 3.5 `McpBuild` — compiler / migrer (synchrone)

| Champ | Valeur |
|-------|--------|
| **Name** | `McpBuild` |
| **IsAsync** | `true` |
| **ReturnValue** | `System.Object` |
| **Parameters** | *(aucun)* |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums.Process;       // HotReloadStepState
using VPSoft.Utils.Helpers.Process;      // HotReloadStepByStepProcess + ProcessUserService
using VPSoft.Domain.Utils.Process.Utils; // ProcessState, ProcessType
```

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

// Empeche les builds concurrents
if (ProcessUserService.HasSameProcessRunningOnGlobal(ProcessType.StepByStep))
    return new { success = false, message = "Un build est deja en cours." };

string processToken = Guid.NewGuid().ToString("N");
var process = new HotReloadStepByStepProcess(processToken, HotReloadStepState.Finalizing, HotReloadStepState.Error);

try
{
    // Synchrone pour le MCP : on attend la fin du build.
    await sm.FormBuilderService.BuildVersion(processToken, process);
}
catch (Exception ex)
{
    return new { success = false, processToken = processToken, message = "Build KO : " + ex.Message };
}

var state = ProcessUserService.GetState(processToken);
var progress = ProcessUserService.GetProgress(processToken);

return new
{
    success = state != ProcessState.Fail,
    processToken = processToken,
    state = state.ToString(),
    progress = progress,
    errors = process.ErrorLogs
};
```

**Notes :**
- Namespaces confirmés : `HotReloadStepByStepProcess` et `ProcessUserService` →
  `VPSoft.Utils.Helpers.Process` ; `ProcessState` / `ProcessType` →
  `VPSoft.Domain.Utils.Process.Utils` ; `HotReloadStepState` → `VPSoft.Domain.Enums.Process`.
- `BuildVersion` lance son propre `Task.Run` + scope DI ; l'`await` ici bloque l'appel MCP
  jusqu'à la fin du build, ce qui donne un retour synchrone exploitable par le MCP.
- Alternative non bloquante : ne pas `await`, renvoyer le `processToken`, puis créer une 4e
  fonction `McpBuildStatus(string processToken)` qui renvoie `ProcessUserService.GetState/GetProgress`.

---

### 3.6 `McpCreateReferenceField` — champ Entity + référence inverse

Crée un champ de type `Entity` côté table courante **et** la propriété inverse côté entité
cible (la relation miroir, ex. `Vehicule.Proprietaire` ↔ `User.Vehicules`).
Reprend exactement le bloc « inverse reference » de `FormBuilderController.CreateDynamicField`
(l.1503-1551).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateReferenceField` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string dynamicSettingsId, string fieldName, string entityNameSelected, string referenceRelationSelected, string inverseReferencePropertyName` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums;
using VPSoft.Domain.Contracts.DynamicFields;
using VPSoft.Domain.Utils.Builder.Descriptor;
using VPSoft.Domain.Utils.Builder.Helper;   // BuilderConfiguration.InheritedClassSuffix
using VPSoft.Domain.Helpers;
using VPSoft.Domain.Helpers.Extensions;
```

> `referenceRelationSelected` : `ReferenceRelation` côté courant
> (`Reference=0`, `HasMany=...`). La relation inverse est déduite automatiquement
> (`Reference` ↔ `HasMany`).

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

// Params MCP => toujours en string, on parse ici
int referenceRelationSelectedInt = string.IsNullOrWhiteSpace(referenceRelationSelected) ? 0 : int.Parse(referenceRelationSelected);

Guid dsGuid = Guid.Parse(dynamicSettingsId);
string tableName = sm.DynamicSettingsService.GetSingleSelect(x => x.Id == dsGuid, x => x.EntityName);
if (string.IsNullOrEmpty(tableName))
    return new { success = false, message = "DynamicSettings introuvable : " + dynamicSettingsId };

// --- Champ de reference (cote courant) ---
var fp = new FormProperty();
fp.Name = fieldName;
fp.Category = FormPropertyCategory.Standard;
fp.Status = FormPropertyStatus.ADDED;
fp.Type = FormPropertyType.Entity;

Type entityTypeFormBuilder = ReflectionHelper.GetTypeEntity(tableName).GetEntityTypeFormBuilderWhenIsManaged();
fp.TableName = sm.SessionFactoryService.GetTableName(entityTypeFormBuilder);

fp.EntityNameSelected = entityNameSelected;
fp.ReferenceRelationSelected = referenceRelationSelectedInt;
fp.InverseReferencePropertyName = string.IsNullOrEmpty(inverseReferencePropertyName) ? null : inverseReferencePropertyName;

Type entityTypeSelected = ReflectionHelper.GetTypeEntity(fp.EntityNameSelected);
fp.EntityFullName = entityTypeSelected.FullName;
fp.ReferenceRelation = (ReferenceRelation)referenceRelationSelectedInt;
fp.EntityTableName = sm.SessionFactoryService.GetTableName(entityTypeSelected);
fp.Nullable = true;

string errorValidationMessage;
if (!fp.ValidateConstraintForCreate(tableName, out errorValidationMessage))
    return new { success = false, message = errorValidationMessage };

var fcd = sm.ConfigurationDescriptorService
    .GetOrCreateInheritedConfigurationDescriptorFromType(ReflectionHelper.GetTypeEntity(tableName));

if (sm.DynamicFieldService.IsPropertyNameExists(fcd, fp, tableName))
    return new { success = false, message = "Le champ existe deja : " + tableName + "." + fieldName };

var model = new FormPropertyModel
{
    DynamicSettingsId = dsGuid,
    TableName = tableName,
    Property = fp,
    DynamicFieldConfigFormModel = new DynamicFieldConfigFormModel()
};

var newField = sm.ConfigurationDescriptorService.AddPropertyWorkflow(model, fcd, fp, out _);
if (newField == null)
    return new { success = false, message = "Echec creation du champ de reference" };

// --- Champ inverse (cote entite cible) ---
object inverseInfo = null;
if (fp.InverseReferencePropertyName != null)
{
    var inverse = new FormProperty();
    inverse.Name = fp.InverseReferencePropertyName;
    inverse.Type = FormPropertyType.Entity;
    inverse.Category = FormPropertyCategory.Standard;
    inverse.Nullable = true;
    inverse.HasDefaultValue = false;
    inverse.IsUnique = false;
    inverse.InverseReferencePropertyName = fp.Name;
    inverse.InverseReference = true;
    inverse.ReferenceRelation = fp.ReferenceRelation == ReferenceRelation.HasMany
        ? ReferenceRelation.Reference
        : (fp.ReferenceRelation == ReferenceRelation.Reference
            ? ReferenceRelation.HasMany
            : fp.ReferenceRelation);
    inverse.EntityFullName = entityTypeFormBuilder.FullName;
    inverse.EntityTableName = fp.TableName;
    inverse.TableName = fp.EntityTableName;

    if (!inverse.ValidateConstraintForCreate(fp.EntityName, out errorValidationMessage))
        return new { success = false, message = "Inverse : " + errorValidationMessage };

    string inverseEntityName = fp.EntityName;
    Type inverseEntityType = ReflectionHelper.GetTypeEntity(inverseEntityName);
    if (inverseEntityName.EndsWith(BuilderConfiguration.InheritedClassSuffix))
        inverseEntityType = inverseEntityType.BaseType;

    var inverseFcd = sm.ConfigurationDescriptorService
        .GetOrCreateInheritedConfigurationDescriptorFromType(inverseEntityType);

    if (sm.DynamicFieldService.IsPropertyNameExists(inverseFcd, inverse, inverseEntityName))
        return new { success = false, message = "Le champ inverse existe deja : " + inverseEntityName + "." + inverse.Name };

    var newInverse = sm.ConfigurationDescriptorService.AddPropertyWorkflow(new FormPropertyModel(), inverseFcd, inverse, out _);
    inverseInfo = new
    {
        fieldId = newInverse != null ? (Guid?)newInverse.Id : null,
        name = inverse.Name,
        table = inverseEntityName
    };
}

return new
{
    success = true,
    fieldId = newField.Id,
    table = tableName,
    fieldName = fieldName,
    inverse = inverseInfo,
    message = "Champ de reference cree (config). Lancer McpBuild."
};
```

**Notes :**
- `fp.EntityName` est dérivé de `EntityFullName` (dernier segment), donc disponible dès que
  `EntityFullName` est renseigné.
- Le suffixe `Extension` (`BuilderConfiguration.InheritedClassSuffix`) gère le cas des entités
  héritées : on remonte sur la classe de base pour la propriété inverse.
- Les deux propriétés ne sont matérialisées qu'au prochain `McpBuild`.

---

### 3.7 Build non-bloquant — `McpBuildAsync` + `McpBuildStatus`

Alternative à `McpBuild` (§3.5) quand on ne veut pas bloquer l'appel MCP pendant toute la
durée du build : `McpBuildAsync` démarre le build et renvoie immédiatement le `processToken`,
puis `McpBuildStatus` est appelé en boucle pour suivre l'avancement.

`ProcessState` : `Done`, `Fail`, `Running`, `Waiting` (`VPSoft.Domain.Utils.Process.Utils`).

#### `McpBuildAsync`

| Champ | Valeur |
|-------|--------|
| **Name** | `McpBuildAsync` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | *(aucun)* |
| **CodeUsing** | `using VPSoft.Domain.Enums.Process;`<br>`using VPSoft.Utils.Helpers.Process;`<br>`using VPSoft.Domain.Utils.Process.Utils;` |

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

if (ProcessUserService.HasSameProcessRunningOnGlobal(ProcessType.StepByStep))
    return new { success = false, message = "Un build est deja en cours." };

string processToken = Guid.NewGuid().ToString("N");
var process = new HotReloadStepByStepProcess(processToken, HotReloadStepState.Finalizing, HotReloadStepState.Error);

// Fire-and-forget : BuildVersion gere son propre Task.Run + scope DI.
_ = sm.FormBuilderService.BuildVersion(processToken, process);

return new
{
    success = true,
    processToken = processToken,
    state = "Running",
    message = "Build lance. Interroger McpBuildStatus avec ce processToken."
};
```

#### `McpBuildStatus`

| Champ | Valeur |
|-------|--------|
| **Name** | `McpBuildStatus` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string processToken` |
| **CodeUsing** | `using VPSoft.Utils.Helpers.Process;`<br>`using VPSoft.Domain.Utils.Process.Utils;` |

```csharp
var state = ProcessUserService.GetState(processToken);
var progress = ProcessUserService.GetProgress(processToken);

if (state == null)
    return new { success = false, processToken = processToken, message = "processToken inconnu ou expire." };

return new
{
    success = state != ProcessState.Fail,
    processToken = processToken,
    state = state.ToString(),       // Done | Fail | Running | Waiting
    progress = progress,            // 0..100
    finished = state == ProcessState.Done || state == ProcessState.Fail
};
```

**Notes :**
- `GetState(string)` → `ProcessState?`, `GetProgress(string)` → `int?`
  (`ProcessUserService`, `VPSoft.Utils.Helpers.Process`).
- Boucle de polling côté MCP : rappeler `McpBuildStatus(processToken)` jusqu'à
  `finished == true`. `state == "Done"` = succès, `"Fail"` = échec.
- `McpBuild` (§3.5) reste préférable si l'orchestration MCP tolère un appel bloquant
  (un seul aller-retour, retour synchrone).

---

### 3.8 Droits d'accès à l'API REST de données (post-build)

Après un `McpBuild` réussi, la table existe en base mais **n'est pas exploitable via l'API REST
dynamique** (`/api/v1/...`, celle utilisée par les outils MCP `create_entity` /
`get_many_select`). L'UI (`AddDynamicEntity`) crée automatiquement les droits ; le chemin
FormBuilder-via-MCP **ne le fait pas**. Il faut donc octroyer **deux couches de permissions**.

#### Les deux couches

| Couche | Entité | Effet si absente |
|--------|--------|------------------|
| 1. Permission d'entité | `RolePermission` (CanSee/CanAdd/CanEdit/CanDelete) | L'API renvoie **HTTP 500** |
| 2. Exposition des champs | `DynamicFieldRole.Api = true` | Les champs sont **silencieusement ignorés** (valeurs `null`, `Code` auto-GUID) |

- **Mécanique couche 2** : `DynamicApiService.GetUsableTable<TEntity>()` filtre les colonnes
  exploitables via `.Where(x => x.Api)`. Un champ dont le `DynamicFieldRole.Api` vaut `false`
  (la valeur par défaut) est exclu du schéma lu/écrit par l'API. Le POST
  (`DynamicApiService.Post` → `CreateImportApi` → `apiImport.Execute()`) n'importe que ces
  colonnes utilisables.
- **Rôle utilisé** : les deux fonctions résolvent le `RoleInModule` de l'**utilisateur courant
  du MCP** (`UserService.GetCurrent()`), via `RoleInModuleService.GetUserRoleInModuleFromEntity`
  (dérivé de l'entité), avec repli sur `DynamicHelper.GetRoleInModuleFromModule(user, moduleId)`.
- **Cache** : `RolePermission` et `DynamicFieldRole` portent `[EntityCache]` ; la sauvegarde
  rafraîchit automatiquement le cache (`ContextInterceptor`), donc les droits sont effectifs
  immédiatement.

> ⚠️ **N'exposer que les champs métier réels.** Exposer **tous** les champs (y compris les
> champs système/hérités/calculés : `Id`, `EntityState`, `CreatedDate`, `AuditDataList`, …)
> casse l'importeur POST (**HTTP 500**). D'où l'approche **liste blanche** de `McpSetFieldsApi`.

#### `McpGrantTablePermission` — octroyer la permission d'entité (couche 1)

| Champ | Valeur |
|-------|--------|
| **Name** | `McpGrantTablePermission` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string entityName, string moduleId` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Models.Entities.Roles;
using VPSoft.Domain.Enums;
using VPSoft.Utils.Helpers;
```

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, message = "entityName requis" };
if (string.IsNullOrWhiteSpace(moduleId)) return new { success = false, message = "moduleId requis" };
Guid moduleGuid = Guid.Parse(moduleId);
var crtUser = sm.UserService.GetCurrent();
if (crtUser == null) return new { success = false, message = "no current user" };
var roleInModule = DynamicHelper.GetRoleInModuleFromModule(crtUser, moduleGuid);
if (roleInModule == null) return new { success = false, message = "current user has no role in module " + moduleId };
var existing = sm.RolePermissionService.GetSingle(x => x.EntityName == entityName && x.ViewName == null && x.Role.Id == roleInModule.Id);
if (existing != null) {
    existing.CanSee = true; existing.CanAdd = true; existing.CanEdit = true; existing.CanDelete = true;
    existing.PermissionType = PermissionType.Write;
    sm.RolePermissionService.Edit(existing);
    return new { success = true, action = "updated", rolePermissionId = existing.Id, roleId = roleInModule.Id, entityName = entityName };
}
var rp = new RolePermission();
rp.Role = roleInModule;
rp.ControllerName = MvcHelper.GetCorrectControllerName(entityName);
rp.PermissionType = PermissionType.Write;
rp.EntityName = entityName;
rp.CanSee = true; rp.CanAdd = true; rp.CanEdit = true; rp.CanDelete = true;
bool ok = sm.RolePermissionService.Create(rp);
return new { success = ok, action = "created", rolePermissionId = rp.Id, roleId = roleInModule.Id, entityName = entityName };
```

**Notes :**
- `ViewName == null` cible la permission de l'entité de base (pas une vue spécifique).
- Idempotent : met à jour la permission existante si elle existe déjà, sinon la crée.

#### `McpSetFieldsApi` — exposer les champs à l'API par liste blanche (couche 2)

Met `DynamicFieldRole.Api = true` pour les champs nommés (liste blanche CSV) et `Api = false`
pour tous les autres. Crée le `DynamicFieldRole` si absent (cas d'une table fraîchement buildée :
`McpCreateField` ne crée **jamais** de `DynamicFieldRole`).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldsApi` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string entityName, string moduleId, string fieldNamesCsv` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums;
using VPSoft.Utils.Helpers;
```

> `DynamicFieldRole`, `EntityState`, et les enums `DynamicFieldRoleActionForAdd/Edit/List`
> sont dans les usings par défaut (`VPSoft.Domain.Models.Builder`).

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName))
        return new { success = false, step = "args", message = "entityName requis" };

    var whitelist = (fieldNamesCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();

    var crtUser = sm.UserService.GetCurrent();
    var role = sm.RoleInModuleService.GetUserRoleInModuleFromEntity(crtUser, entityName);
    if (role == null && !string.IsNullOrWhiteSpace(moduleId))
        role = DynamicHelper.GetRoleInModuleFromModule(crtUser, Guid.Parse(moduleId));
    if (role == null)
        return new { success = false, step = "role", message = "Aucun RoleInModule resolu" };

    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    if (dsId == Guid.Empty)
        return new { success = false, step = "ds", message = "DynamicSettings introuvable" };
    var ds = sm.DynamicSettingsService.GetSingle(dsId);

    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();

    var enabled = new List<string>();
    var disabled = new List<string>();

    foreach (var field in fields)
    {
        bool desired = whitelist.Contains(field.PropertyName);
        var dfr = sm.DynamicFieldRoleService.GetDynamicFieldRoleFromProperty(field, null, role.Id, false);

        if (dfr == null)
        {
            if (!desired) continue;
            dfr = new DynamicFieldRole
            {
                EntityState = EntityState.Active,
                DynamicField = field,
                RoleInModule = role,
                DynamicSettings = ds,
                CanRead = true,
                RoleActionForAdd = DynamicFieldRoleActionForAdd.Active,
                RoleActionForEdit = DynamicFieldRoleActionForEdit.Active,
                Table = DynamicFieldRoleActionForList.Display,
                Api = true
            };
            sm.DynamicFieldRoleService.Create(dfr);
            enabled.Add(field.PropertyName);
        }
        else if (desired)
        {
            dfr.Api = true;
            dfr.CanRead = true;
            if (dfr.RoleActionForAdd == DynamicFieldRoleActionForAdd.Inactive) dfr.RoleActionForAdd = DynamicFieldRoleActionForAdd.Active;
            if (dfr.RoleActionForEdit == DynamicFieldRoleActionForEdit.Inactive) dfr.RoleActionForEdit = DynamicFieldRoleActionForEdit.Active;
            sm.DynamicFieldRoleService.Edit(dfr);
            enabled.Add(field.PropertyName);
        }
        else if (dfr.Api)
        {
            dfr.Api = false;
            sm.DynamicFieldRoleService.Edit(dfr);
            disabled.Add(field.PropertyName);
        }
    }

    return new { success = true, entityName = entityName, roleId = role.Id, enabled = enabled, disabled = disabled, message = "Api flags mis a jour (whitelist)." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

**Notes :**
- `GetDynamicFieldRoleFromProperty(field, viewName:null, roleId, fromCache:false)` renvoie la
  ligne `DynamicFieldRole` propre au rôle pour cette propriété (ou `null` si elle n'existe pas).
- `GetDynamicFieldsFromEntity` retourne **tous** les champs (y compris hérités/système/calculés) ;
  la liste blanche garantit qu'on n'expose que les champs métier voulus à l'écriture.
- Validé end-to-end : après `McpSetFieldsApi("…", "…", "Code,Immatriculation,Kilometrage")`,
  `create_entity` persiste correctement les 3 valeurs (avant : `Code` = GUID auto, autres `null`).
- Le `try/catch` interne sert à **remonter les erreurs** : le wrapper de DynamicFunction avale
  sinon les exceptions (`return default(...)` → `null` silencieux).

#### `McpSetFieldsUiVisibility` — visibilité liste / create / edit par liste blanche (couche 3, UI)

`McpSetFieldsApi` (couche 2) règle uniquement l'**exposition à l'API d'écriture** (`Api`).
La **visibilité dans l'UI** (vue liste, formulaire de création, formulaire d'édition) est
gérée par d'**autres** propriétés du `DynamicFieldRole`, propres au rôle :

| Propriété `DynamicFieldRole` | Vue concernée | Consommateur autoritaire |
|------------------------------|---------------|--------------------------|
| `Table` (`DynamicFieldRoleActionForList`) | **liste** | `GetDynamicFieldsByAction(FieldAction.Table)` → filtre `x.Table != Inactive` |
| `RoleActionForAdd` (`…ForAdd`) | **create** | `GetDynamicFieldsByAction(FieldAction.Create)` → filtre `RoleActionForAdd != Inactive` |
| `RoleActionForEdit` (`…ForEdit`) | **edit** | `GetDynamicFieldsByAction(FieldAction.Edit)` → filtre `RoleActionForEdit != Inactive` |
| `CanRead` | **detail** | `GetDynamicFieldsByAction(FieldAction.Detail)` → filtre `x.CanRead` |

Le mapping autoritaire est `SaveDynamicSettingsRolesPermissions` (l.3420 de
`DynamicSettingsService`, déclenché par `/api/FormBuilder/SaveDynamicSettingsConfig`) :
`dynamicFieldRole.Table = fieldRoleModel.RoleActionForTable`, etc. Cette fonction reproduit
le même effet par programmation, sans passer par le payload complet de l'UI.

Met `Table`/`RoleActionForAdd`/`RoleActionForEdit` à **actif** pour les champs de la liste
blanche, **Inactive** pour tous les autres — **en préservant le flag `Api`**. Un champ peut
ainsi rester **écrivable via l'API tout en étant masqué de l'UI** (cas `Kilometrage` :
`Api=true` mais absent de la liste / des formulaires).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldsUiVisibility` |
| **IsAsync** | `false` |
| **ReturnValue** | `System.Object` |
| **Parameters** | `string entityName, string moduleId, string visibleFieldsCsv` |
| **CodeUsing** | voir bloc ci-dessous |

```csharp
// CodeUsing
using VPSoft.Domain.Enums;
using VPSoft.Utils.Helpers;
```

**CodeFunction :**

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName))
        return new { success = false, step = "args", message = "entityName requis" };

    var whitelist = (visibleFieldsCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();

    var crtUser = sm.UserService.GetCurrent();
    var role = sm.RoleInModuleService.GetUserRoleInModuleFromEntity(crtUser, entityName);
    if (role == null && !string.IsNullOrWhiteSpace(moduleId))
        role = DynamicHelper.GetRoleInModuleFromModule(crtUser, Guid.Parse(moduleId));
    if (role == null)
        return new { success = false, step = "role", message = "Aucun RoleInModule resolu" };

    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    if (dsId == Guid.Empty)
        return new { success = false, step = "ds", message = "DynamicSettings introuvable" };
    var ds = sm.DynamicSettingsService.GetSingle(dsId);

    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();

    var shown = new List<string>();
    var hidden = new List<string>();

    foreach (var field in fields)
    {
        bool desired = whitelist.Contains(field.PropertyName);
        var dfr = sm.DynamicFieldRoleService.GetDynamicFieldRoleFromProperty(field, null, role.Id, false);

        if (dfr == null)
        {
            dfr = new DynamicFieldRole
            {
                EntityState = EntityState.Active,
                DynamicField = field,
                RoleInModule = role,
                DynamicSettings = ds,
                CanRead = true,
                Api = false,
                Table = desired ? DynamicFieldRoleActionForList.Display : DynamicFieldRoleActionForList.Inactive,
                RoleActionForAdd = desired ? DynamicFieldRoleActionForAdd.Active : DynamicFieldRoleActionForAdd.Inactive,
                RoleActionForEdit = desired ? DynamicFieldRoleActionForEdit.Active : DynamicFieldRoleActionForEdit.Inactive
            };
            sm.DynamicFieldRoleService.Create(dfr);
        }
        else
        {
            dfr.Table = desired ? DynamicFieldRoleActionForList.Display : DynamicFieldRoleActionForList.Inactive;
            dfr.RoleActionForAdd = desired ? DynamicFieldRoleActionForAdd.Active : DynamicFieldRoleActionForAdd.Inactive;
            dfr.RoleActionForEdit = desired ? DynamicFieldRoleActionForEdit.Active : DynamicFieldRoleActionForEdit.Inactive;
            if (desired) dfr.CanRead = true;
            // NB : on ne touche PAS dfr.Api → l'exposition API reste intacte.
            sm.DynamicFieldRoleService.Edit(dfr);
        }

        (desired ? shown : hidden).Add(field.PropertyName);
    }

    return new { success = true, entityName = entityName, roleId = role.Id, shown = shown, hidden = hidden, message = "Visibilite UI (liste/create/edit) mise a jour (whitelist)." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

**Notes :**
- Distinct de `McpSetFieldsApi` : ici on règle la **visibilité UI** (liste/create/edit), pas
  l'exposition API. Les deux fonctions sont complémentaires et **n'interfèrent pas** (le flag
  `Api` est explicitement préservé).
- `GetDynamicFieldRolesFromEntity` (branche entité de base, `viewName == null`) ne renvoie que
  les lignes du rôle courant ; créer/mettre à jour les `DynamicFieldRole` du rôle suffit donc à
  restreindre liste et formulaires pour ce rôle.
- Validé : `McpSetFieldsUiVisibility("McpDemoVehicule", "<moduleId>", "Code,Immatriculation")`
  → `shown=[Code, Immatriculation]`, `hidden=[Kilometrage, …]`. Vérifié en base :
  `Code`/`Immatriculation` ont `Table=1`/`Add=1`/`Edit=1` ; `Kilometrage` a `Table=0`/`Add=0`/`Edit=0`
  mais `Api=true` (préservé) ; tous les autres `Table=0`/`Add=0`/`Edit=0`.

---

## 3.9 Modèle de données complet — types de champs avancés & configuration

Cette section couvre **l'ensemble des capacités** nécessaires pour bâtir un modèle de données
complet via MCP : champ calculé (expression C# **privilégiée**, formule SQL **déconseillée**),
lien inverse (reverse-link), labels multilingues, descriptions, unités, exposition export,
options de rendu, tableaux imbriqués. Toutes les fonctions ci-dessous ont été **créées et
validées de bout en bout** sur l'entité de démo `McpDemoVehicule` (+ entité enfant
`McpDemoIntervention`).

### 3.9.0 Récapitulatif des fonctions ajoutées

| Fonction | Rôle | Rebuild ? |
|----------|------|-----------|
| `McpCreateReverseLink` | Champ entité + propriété inverse (collection ↔ référence) | **Oui** |
| `McpSetFieldExpression` | Corps **C#** d'un champ expression (PRIVILÉGIÉ) | **Oui** |
| `McpCreateFormulaField` / `McpSetFieldFormula` | Champ **formule SQL** (DÉCONSEILLÉ) | **Oui** |
| `McpSetFieldLabel` | Labels multilingues (`DynamicFieldName`) | Non |
| `McpSetFieldDescription` | Descriptions multilingues (`DynamicFieldDescription`) | Non |
| `McpSetFieldUnit` | Unité sur un champ numérique (`FormProperty.Unit`) | **Oui** (à chaque pose/changement d'unité) |
| `McpSetFieldExport` | Exposition export (`DynamicFieldRole.ImportExport`) | Non |
| `McpSetFieldRender` | Rendu global (`RenderTypeOverrided`, `FieldDisplay`, `DontDisplayLabel`) | Non |
| `McpSetFieldNestedTable` | Tableau imbriqué (`RenderType.Table` + `EntityOrViewForNestedTable`) | Non |

> **Règle de séquencement schéma vs config.** Les opérations qui **changent le schéma**
> (nouvelle colonne / nouvelle relation : `McpCreateField`, `McpCreateReverseLink`,
> expression/formule, **1ʳᵉ unité**) doivent être suivies d'un `McpBuild`. Les opérations de
> **configuration pure** (labels, descriptions, export, rendu, nested table, visibilité,
> Api) prennent effet **immédiatement** via le cache `[EntityCache]`, sans rebuild.
> Important : une entité enfant doit être **buildée** avant qu'un reverse-link puisse la cibler
> (le forward résout `ReflectionHelper.GetTypeEntity(target)` au moment de la config).

### 3.9.1 Champ expression **C#** — `McpSetFieldExpression` (PRIVILÉGIÉ)

C'est l'approche recommandée pour un champ calculé. Le générateur
(`ClassBuilder.cs:153-180`) émet, pour un `FormPropertyCategory.ExpressionField` :

```csharp
private int _kilometrageTotal;
private int KilometrageTotalFunction(int value) {
    <ReferenceLambda>            // ClassBuilder.cs:168 — DOIT contenir "return", sinon throw généré
}
public virtual int KilometrageTotal {
    get { return _kilometrageTotal; }
    set { _kilometrageTotal = KilometrageTotalFunction(value); /* ... */ }
}
```

> ⚠️ **`ReferenceLambda` est un CORPS de méthode C#, pas un lambda `x => …`.** Il doit contenir
> `return` et référence directement les propriétés de l'entité courante. Un `"x => x.A + x.B"`
> (sans `return`) génère un `throw` → la valeur reste `null`. Le bon contenu est
> `"return Kilometrage + KilometrageAnnuel;"`. Le recalcul se fait côté C# au save/load via
> `RefreshExpressionFields()` (`ClassBuilder.cs:372-385`, qui réaffecte `Prop = Prop;`). Mapping :
> `Map(x => x.Prop, "<col>").Access.CamelCaseField(...)` (`ClassMappingBuilder.cs:188-190`).

Création en deux temps : `McpCreateExpressionField` (§3.4, crée le champ avec un type de retour),
puis **`McpSetFieldExpression`** pour (re)définir le corps C# :

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldExpression` |
| **Parameters** | `string dynamicFieldId, string referenceLambda` |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Utils.Builder.Descriptor;` · `using VPSoft.Domain.Repositories;` |

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try
{
    if (string.IsNullOrWhiteSpace(dynamicFieldId)) return new { success = false, step = "args", message = "dynamicFieldId requis" };
    if (string.IsNullOrWhiteSpace(referenceLambda)) return new { success = false, step = "args", message = "referenceLambda requis (corps C# avec 'return')" };
    if (!referenceLambda.Contains("return")) return new { success = false, step = "validate", message = "Le corps doit contenir 'return' (corps de methode C#, pas un lambda x => ...)." };
    Guid fid = Guid.Parse(dynamicFieldId);
    var fp = rm.DynamicFieldRepository.GetSingleSelect(x => x.Id == fid, x => x.FormProperty);
    if (fp == null) return new { success = false, step = "find", message = "FormProperty introuvable : " + dynamicFieldId };
    fp.ReferenceLambda = referenceLambda;
    fp.Status = FormPropertyStatus.MODIFIED;
    rm.FormPropertyRepository.Edit(fp);
    return new { success = true, fieldId = fid, referenceLambda = fp.ReferenceLambda, message = "Expression C# mise a jour. Lancer McpBuild." };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

**Validé** : `KilometrageTotal` avec `"return Kilometrage + KilometrageAnnuel;"` → après `McpBuild`
et création d'un enregistrement `{Kilometrage:200000, KilometrageAnnuel:20000}`,
`KilometrageTotal = 220000`. (Note : le recalcul se fait au save ; les enregistrements créés
**avant** la définition de l'expression doivent être re-sauvegardés pour se recalculer.)

### 3.9.2 Champ **formule SQL** — `McpCreateFormulaField` / `McpSetFieldFormula` (DÉCONSEILLÉ)

Alternative SQL : `FormPropertyCategory.Formula` → mapping `Map(x => x.Prop).Formula(@"(<sql>)")`
(`ClassMappingBuilder.cs:177-178`) = colonne calculée en base. **Déconseillée** (couplage SQL,
dépend du dialecte, nécessite les noms de colonnes physiques).

> ⚠️ **La formule référence les noms de colonnes PHYSIQUES**, préfixés `FormBuilder_`
> (ex. `FormBuilder_Kilometrage + FormBuilder_KilometrageAnnuel`), **pas** les noms de propriété.
> Une formule sur les noms de propriété produit `could not execute query` (colonne inexistante).

`McpCreateFormulaField(dynamicSettingsId, fieldName, returnType, formula, formulaSqlite)` crée le
champ ; `McpSetFieldFormula(dynamicFieldId, formula, formulaSqlite)` édite la formule d'un champ
existant (puis `McpBuild`). **Validé** : `KilometrageCumul = 220000`.

### 3.9.3 Lien inverse (reverse-link) — `McpCreateReverseLink`

Réplique la logique du contrôleur (`FormBuilderController.cs:1428-1551`) : crée le champ **forward**
(`FormPropertyType.Entity` + `ReferenceRelation` + `InverseReferencePropertyName`) **et** la
propriété **inverse** sur l'entité cible (relation inversée : `HasMany`↔`Reference`,
`InverseReference = true`, câblage croisé `EntityFullName`/`EntityTableName`/`TableName`).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpCreateReverseLink` |
| **Parameters** | `string entityName, string targetEntityName, string fieldName, string referenceRelationSelected, string inverseReferencePropertyName` |
| **CodeUsing** | `VPSoft.Domain.Enums` · `VPSoft.Domain.Contracts.DynamicFields` · `VPSoft.Domain.Utils.Builder.Descriptor` · `VPSoft.Domain.Helpers` · `VPSoft.Domain.Helpers.Extensions` · `VPSoft.Domain.Utils.Builder.Helper` |

`ReferenceRelation` : `Reference=0, HasMany=1, HasManyToMany=2, HasOne=3`. **Validé** :
`McpCreateReverseLink("McpDemoVehicule","McpDemoIntervention","Interventions","1","Vehicule")`
→ `McpDemoVehicule.Interventions` (HasMany) ↔ `McpDemoIntervention.Vehicule` (Reference). Le code
complet de la fonction est reproduit dans la base (folder API MCP).

### 3.9.4 Labels multilingues — `McpSetFieldLabel`

Les libellés ne sont **pas** sur `DynamicField` : entité dédiée **`DynamicFieldName`**
(`Name`, `ColumnName`, `Tooltip`, `Culture`), une ligne par culture. La fonction crée/édite ces
lignes pour chaque culture fournie.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldLabel` |
| **Parameters** | `string entityName, string fieldName, string labels` (format `fr-FR=Libellé\|en-US=Label`) |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Models.Entities;` |

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();
    var field = fields.FirstOrDefault(f => f.PropertyName == fieldName);
    if (field == null) return new { success = false, step = "field", message = "Champ introuvable : " + fieldName };
    var pairs = (labels ?? "").Split('|').Select(s => s.Trim()).Where(s => s.Contains('=')).ToList();
    var applied = new List<object>();
    foreach (var p in pairs)
    {
        int idx = p.IndexOf('='); string code = p.Substring(0, idx).Trim(); string val = p.Substring(idx + 1).Trim();
        var culture = sm.CultureService.GetSingle(code);
        if (culture == null) { applied.Add(new { culture = code, status = "culture introuvable" }); continue; }
        var existing = sm.DynamicFieldNameService.FindDynamicFieldName(x => x.DynamicField.Id == field.Id && x.Culture.CultureCode == code);
        if (existing != null) { existing.Name = val; sm.DynamicFieldNameService.Edit(existing); applied.Add(new { culture = code, label = val, status = "updated" }); }
        else { var dfn = new DynamicFieldName { EntityState = EntityState.Active, Name = val, Culture = culture, DynamicField = field }; sm.DynamicFieldNameService.Create(dfn); applied.Add(new { culture = code, label = val, status = "created" }); }
    }
    return new { success = true, entityName = entityName, fieldName = fieldName, applied = applied };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

**Validé** : `Kilometrage` → `fr-FR="Kilométrage"`, `en-US="Mileage"` (lignes `DynamicFieldName`
vérifiées). Le flag `DynamicField.IsTranslate` (= `properties.translation`) reste distinct : il
concerne la **traductibilité du contenu** du champ, pas son libellé.

### 3.9.5 Descriptions multilingues — `McpSetFieldDescription`

Entité dédiée **`DynamicFieldDescription`** (`Description`, `Culture`). Même schéma que les labels
(`Find`/`Create`/`Edit` par culture). Service de raccourci natif : `SaveDynamicFieldDescription(
FieldDescriptionFormModel{ DynamicFieldId, CultureParameters })`.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldDescription` |
| **Parameters** | `string entityName, string fieldName, string descriptions` (format `fr-FR=…\|en-US=…`) |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Models.Entities;` |

Corps identique à `McpSetFieldLabel` mais sur `DynamicFieldDescriptionService.Find/Create/Edit`
avec `new DynamicFieldDescription { EntityState = EntityState.Active, Description = val, Culture =
culture, DynamicField = field }`. **Validé** : descriptions fr-FR/en-US sur `Kilometrage`.

### 3.9.6 Unité — `McpSetFieldUnit`

L'unité est portée par **`DynamicField.FormProperty.Unit`** (référence vers l'entité `Unit`,
`UnitType` : Currency/Length/Mass/Surface/Volume/Speed/Energy/Temperature…). Logique reprise de
`DynamicFieldService.cs:3128-3161`.

> ⚠️ **Rebuild OBLIGATOIRE à chaque pose/changement d'unité.** Le `rebuildRequired` renvoyé par
> `McpSetFieldUnit` ne couvre QUE le besoin **schéma** (relation `DynamicMeasures`,
> `ClassMappingBuilder.cs:293-296`, vrai seulement la **1ʳᵉ unité** de l'entité). Mais le **formatage
> avec l'unité est généré au build** (champs virtuels « formatés », cf. `ClassBuilder`/`DynamicFieldVirtualFormattedNumber`)
> et `DynamicField` est `[EntityCache]` (l'édition via repository ne rafraîchit pas le cache). Donc
> **lancer `McpBuild` systématiquement** après un changement d'unité — sinon l'unité n'apparaît pas
> (constaté en prod : ce n'est ni un simple visuel ni juste du cache, c'est du code généré + cache).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldUnit` |
| **Parameters** | `string entityName, string fieldName, string unitId` |
| **CodeUsing** | `VPSoft.Domain.Enums` · `VPSoft.Domain.Utils.Builder.Descriptor` · `VPSoft.Domain.Repositories` · `VPSoft.Domain.Helpers` · `VPSoft.Domain.Helpers.Extensions` |

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try
{
    var field = rm.DynamicFieldRepository.GetSingle(x => x.EntityName == entityName && x.PropertyName == fieldName && x.ViewName == null);
    if (field == null) return new { success = false, step = "field", message = "Champ introuvable" };
    var fp = field.FormProperty;
    if (fp == null) return new { success = false, step = "formProperty", message = "FormProperty null" };
    Guid uguid = Guid.Parse(unitId);
    var unit = sm.UnitService.GetSingle(x => x.Id == uguid);
    if (unit == null) return new { success = false, step = "unit", message = "Unit introuvable" };
    bool entityIsWithUnit = ReflectionHelper.GetTypeEntity(entityName).IsWithUnitEntity();
    bool unitAdded = fp.Unit == null;
    fp.Unit = unit;
    bool rebuildRequired = false;
    if (unitAdded && !entityIsWithUnit && fp.Status == FormPropertyStatus.OK)
    { fp.Status = FormPropertyStatus.MODIFIED; field.Status = FormPropertyStatus.MODIFIED; rebuildRequired = true; rm.DynamicFieldRepository.Edit(field); }
    rm.FormPropertyRepository.Edit(fp);
    return new { success = true, entityName, fieldName, unit = unit.Name, rebuildRequired, message = rebuildRequired ? "Unite ajoutee. REBUILD REQUIS. Lancer McpBuild." : "Unite mise a jour (pas de rebuild)." };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

**Validé** : unité (type Length) posée sur `Kilometrage`, `rebuildRequired=true`, après `McpBuild`
`FormProperty.Unit` confirmée.

### 3.9.7 Exposition export — `McpSetFieldExport`

`DynamicFieldRole.ImportExport` (enum `IEDynamicFieldRole` : `NotAvailable=0`, `Both=1`,
`OnlyExport=2`). Même patron liste-blanche que `McpSetFieldsApi`, mais sur `ImportExport`.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldExport` |
| **Parameters** | `string entityName, string moduleId, string exportFieldsCsv` |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Utils.Helpers;` · `using VPSoft.Domain.Models.ImportExport;` |

Met `ImportExport = IEDynamicFieldRole.Both` pour la liste blanche, `NotAvailable` pour les autres
(crée le `DynamicFieldRole` si absent). **Validé** : export activé sur
`Code,Immatriculation,Kilometrage,KilometrageTotal` (`ImportExport=1` en base). Distinct de
l'« exposition interface » qui = `DynamicFieldRole.Api` (§3.8) ; le concept `IEUsableTable`/
`IEType.Interface` est un modèle d'intégration import/export séparé.

### 3.9.8 Options de rendu globales — `McpSetFieldRender`

`DynamicField.RenderTypeOverrided` (enum `RenderType`) pilote le rendu, complété par `FieldDisplay`
(`NewLine=0`/`InSuccession=1`) et `DontDisplayLabel`. Consommé au runtime → **pas de rebuild**.

`RenderType` : `Input=0, Textarea=1, Editor=2, List=3, ListMultiple=4, CheckboxMultiple=5,
RadioButton=6, Color=7, File=8, Date=9, DateTime=10, MultiCulture=12, Password=20, Table=21,
Tree=22, InlineEditor=23` (cf. `RenderType.cs`).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldRender` |
| **Parameters** | `string entityName, string fieldName, string renderType, string dontDisplayLabel, string fieldDisplay` |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Repositories;` |

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
var field = rm.DynamicFieldRepository.GetSingle(x => x.EntityName == entityName && x.PropertyName == fieldName && x.ViewName == null);
if (field == null) return new { success = false, message = "Champ introuvable" };
if (!string.IsNullOrWhiteSpace(renderType)) { RenderType rt; int rti; if (int.TryParse(renderType, out rti)) rt = (RenderType)rti; else rt = (RenderType)Enum.Parse(typeof(RenderType), renderType, true); field.RenderTypeOverrided = rt; }
if (!string.IsNullOrWhiteSpace(dontDisplayLabel)) field.DontDisplayLabel = (dontDisplayLabel == "true" || dontDisplayLabel == "1");
if (!string.IsNullOrWhiteSpace(fieldDisplay)) field.FieldDisplay = (FieldDisplay)Enum.Parse(typeof(FieldDisplay), fieldDisplay, true);
rm.DynamicFieldRepository.Edit(field);
```

**Validé** : `Immatriculation` → `RenderTypeOverrided=1` (Textarea), `FieldDisplay=1` (InSuccession).

### 3.9.9 Tableaux imbriqués (sous-formulaires) — `McpSetFieldNestedTable`

Affiche une collection enfant comme **grille éditable** : `RenderTypeOverrided = RenderType.Table`
(21) + `EntityOrViewForNestedTable = <entité enfant>` + `NestedTableSaveMode` (`Cascade`/`Instantly`)
+ boutons (`ActivateButtonToCreatePropertyData`/`…SelectFromTable`) + `DynamicFieldRole.EditInLine`
pour l'édition en ligne. Pas de rebuild.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldNestedTable` |
| **Parameters** | `string entityName, string moduleId, string collectionFieldName, string childEntityOrView, string saveMode, string editInLine` |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Repositories;` · `using VPSoft.Utils.Helpers;` |

S'applique sur un champ **collection** (typiquement le forward d'un reverse-link). **Validé** sur
`McpDemoVehicule.Interventions` → `RenderTypeOverrided=21`, `EntityOrViewForNestedTable=
McpDemoIntervention`, `Cascade`. NB : `EditInLine` n'est posé que si le `DynamicFieldRole` du champ
existe pour le rôle (le créer au préalable via `McpSetFieldsApi`/`McpSetFieldsUiVisibility`).

### 3.9.10 Disposition du formulaire (sections) — `McpAddFieldsToSection`

> ⚠️ **Deux mécanismes distincts pour « voir » un champ :**
> 1. **Vue liste** → pilotée par `DynamicFieldRole.Table` (§3.8, `McpSetFieldsUiVisibility`).
> 2. **Formulaire Create / Edit / Detail** → pilotée par les **sections** (`DynamicSection`).
>    Un champ **non affecté à une section** apparaît dans « Propriétés sans section » de l'AGL et
>    **ne s'affiche pas dans le formulaire**, même si son `DynamicFieldRole` est actif.

Les champs créés via MCP (`McpCreateField`, expression, reverse-link…) atterrissent **sans
section** → il faut les **affecter à une section** pour qu'ils apparaissent au formulaire. C'est
l'équivalent programmatique du glisser-déposer de l'onglet *Sections* de l'AGL (endpoint
`/api/FormBuilder/SaveDynamicSettingsConfig` → `DynamicSectionService.MoveFieldsToFromSections`,
`DynamicSectionService.cs:438-476`).

**Modèle.** `DynamicSection.DynamicSectionPropertyNames` (ISet de `DynamicSectionPropertyName`
{ `PropertyName`, `OrderBy`, `DynamicSection` }) = la liste ordonnée des champs d'une section.
Affecter un champ = ajouter une ligne `DynamicSectionPropertyName` à la section puis `Edit` (cascade).
Alternative : préciser la section **à la création** du champ (modale AGL → `SectionId`).

| Champ | Valeur |
|-------|--------|
| **Name** | `McpAddFieldsToSection` |
| **Parameters** | `string entityName, string sectionName, string fieldNamesCsv` (`sectionName` = `Code` **ou** libellé, ex. `Formulaire`) |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Models.Builder;` |

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    var fieldsCsv = (fieldNamesCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    var sections = sm.DynamicSectionService.GetDynamicSectionsFromEntity(entityName, null, false).ToList();
    if (!sections.Any()) return new { success = false, step = "sections", message = "Aucune section pour " + entityName };
    var target = sections.FirstOrDefault(s => s.Code == sectionName || s.LocalizedName == sectionName);
    if (target == null) return new { success = false, step = "section", message = "Section introuvable : " + sectionName, available = sections.Select(s => new { s.Code, name = s.LocalizedName }).ToList() };
    var section = sm.DynamicSectionService.GetSingle(x => x.Id == target.Id, x => x.DynamicSectionPropertyNames);
    int order = section.DynamicSectionPropertyNames.Any() ? section.DynamicSectionPropertyNames.Max(x => x.OrderBy) : 0;
    var added = new List<string>(); var skipped = new List<string>();
    foreach (var fn in fieldsCsv)
    {
        if (section.DynamicSectionPropertyNames.Any(x => x.PropertyName == fn)) { skipped.Add(fn); continue; }
        order++;
        section.DynamicSectionPropertyNames.Add(new DynamicSectionPropertyName { DynamicSection = section, PropertyName = fn, OrderBy = order });
        added.Add(fn);
    }
    sm.DynamicSectionService.Edit(section);
    return new { success = true, entityName, section = (string.IsNullOrEmpty(section.LocalizedName) ? section.Code : section.LocalizedName), added, skipped, message = "Champs ajoutes a la section (effet immediat, pas de rebuild)." };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

**Validé** : `McpAddFieldsToSection("McpDemoVehicule","Formulaire","Kilometrage,KilometrageAnnuel,
KilometrageTotal,KilometrageCumul,Interventions")` → section *Formulaire* =
`[Code, Immatriculation, Kilometrage, KilometrageAnnuel, KilometrageTotal, KilometrageCumul,
Interventions]` (ordres 1→7). Effet immédiat (cache) ; **F5** sur le formulaire.

> **Checklist « rendre un champ pleinement visible »** : (1) `McpSetFieldsApi` (écriture API) ·
> (2) `McpSetFieldsUiVisibility` (vue liste + éligibilité create/edit) · (3) **`McpAddFieldsToSection`**
> (présence dans le formulaire). Les trois sont indépendantes.

### 3.9.11 Label + infobulle + description en un appel — `McpSetFieldTexts`

`McpSetFieldLabel` (§3.9.4) ne pose que le `Name`. Pour poser **label + infobulle (Tooltip) +
description** en une fois (par culture), `McpSetFieldTexts` écrit `DynamicFieldName.Name` +
`DynamicFieldName.Tooltip` **et** `DynamicFieldDescription.Description`.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetFieldTexts` |
| **Parameters** | `string entityName, string fieldName, string culture, string label, string tooltip, string description` |
| **CodeUsing** | `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Models.Entities;` |

**Validé** sur les 4 champs de `McpDemoIntervention` (fr-FR) : `Code`/`Libellé`/`Coût (€)`/`Véhicule`
avec infobulles + descriptions (lignes `DynamicFieldName.Tooltip` + `DynamicFieldDescription` vérifiées).

### 3.9.12 Création de données + liens de référence — `McpSetReference`

**Créer un enregistrement** : outil MCP `create_entity`. **⚠️ Convention de référence** : pour
lier un champ entité, passer **le champ directement avec le code** de la cible
(`{"Vehicule": "DEMO-EXPR2"}`), **PAS** le suffixe `_Code_` (`Vehicule_Code_` est resté `null`
à la création sur cette entité). Le code de la cible = sa propriété `IsEntityCode` (`Code`).

**⚠️ Limite `update_entity_patch`** : sur cette entité l'import de **modification** (PATCH) renvoie
**HTTP 500** (peu importe `_Code_` ou champ direct). Pour (re)poser une référence de façon fiable,
utiliser **`McpSetReference`** qui pose le lien **directement en C#** via la session NHibernate
(`NHSessionHelper.GetCurrentSession()`), en contournant l'importeur de modification.

| Champ | Valeur |
|-------|--------|
| **Name** | `McpSetReference` |
| **Parameters** | `string entityName, string entityCode, string referenceFieldName, string targetEntityName, string targetCode` |
| **CodeUsing** | `using VPSoft.Domain.Helpers;` · `using VPSoft.Domain.Helpers.Extensions;` · `using NHibernate;` · `using NHibernate.Criterion;` |

```csharp
var session = NHSessionHelper.GetCurrentSession();
Type srcType = ReflectionHelper.GetTypeEntity(entityName);
Type tgtType = ReflectionHelper.GetTypeEntity(targetEntityName);
var src = session.CreateCriteria(srcType).Add(Restrictions.Eq("Code", entityCode)).UniqueResult();
var tgt = session.CreateCriteria(tgtType).Add(Restrictions.Eq("Code", targetCode)).UniqueResult();
srcType.GetProperty(referenceFieldName).SetValue(src, tgt);   // entité suivie → dirty-check
session.Flush();
```

**Validé** : 5 interventions créées sur `McpDemoIntervention` puis liées —
`DEMO-EXPR2` ← INT-001/002/004/005, `DEMO-EXPR` ← INT-003 (vérifié via `Vehicule.Immatriculation`).
Ces interventions s'affichent dans le **tableau imbriqué** `Interventions` du formulaire véhicule (§3.9.9).

---

## 4. Séquence type via MCP

```
1) McpCreateTable("Vehicule", "EntityAudit", false, true, "<moduleId>", "")
      → { dynamicSettingsId: "..." }
2) McpCreateField("<dynamicSettingsId>", "Immatriculation", 0, true, false, "", 0)   // String
   McpCreateField("<dynamicSettingsId>", "Kilometrage",     1, true, false, "", 0)   // Integer
   McpCreateField("<dynamicSettingsId>", "Proprietaire",    7, true, false, "User", 0) // Entity → User
   // Champ Enum (liste de valeurs) — cf. §3.3
   McpCreateEnumField("<dynamicSettingsId>", "Etat", false, false, "EtatVehicule",
        "[{\"TechnicalName\":\"Neuf\",\"Value\":1,\"IsDefault\":true},{\"TechnicalName\":\"Occasion\",\"Value\":2}]")
   // Champ Expression (calculé) — cf. §3.4
   McpCreateExpressionField("<dynamicSettingsId>", "AgeKm", 1, "x => x.Kilometrage / 1000", "")
3) McpBuild()
      → { success: true, state: "Done" }
4) // Droits API REST (obligatoire pour create_entity / get_many_select) — cf. §3.8
   McpGrantTablePermission("Vehicule", "<moduleId>")
      → { success: true, action: "created" }                       // couche 1 : entité
   McpSetFieldsApi("Vehicule", "<moduleId>", "Immatriculation,Kilometrage,Proprietaire,Etat,AgeKm")
      → { success: true, enabled: [...], disabled: [...] }          // couche 2 : champs
5) // (optionnel) Restreindre la visibilité UI liste/create/edit — cf. §3.8
   McpSetFieldsUiVisibility("Vehicule", "<moduleId>", "Immatriculation,Etat")
      → { success: true, shown: [...], hidden: [...] }              // couche 3 : UI
6) // (optionnel) Modèle avancé — cf. §3.9
   //   schéma (→ McpBuild) :
   McpCreateReverseLink("Vehicule","Intervention","Interventions","1","Vehicule")  // lien inverse
   McpCreateExpressionField("<dsId>","KmTotal","1","","")                          // crée le champ
   McpSetFieldExpression("<fieldId>","return Kilometrage + KilometrageAnnuel;")    // corps C# (privilégié)
   McpSetFieldUnit("Vehicule","Kilometrage","<unitId>")                            // 1ère unité → rebuild
   McpBuild()
   //   config (effet immédiat, sans rebuild) :
   McpSetFieldLabel("Vehicule","Kilometrage","fr-FR=Kilométrage|en-US=Mileage")
   McpSetFieldDescription("Vehicule","Kilometrage","fr-FR=...|en-US=...")
   McpSetFieldExport("Vehicule","<moduleId>","Code,Immatriculation,Kilometrage")
   McpSetFieldRender("Vehicule","Immatriculation","Textarea","false","InSuccession")
   McpSetFieldNestedTable("Vehicule","<moduleId>","Interventions","Intervention","Cascade","true")
```

Après le `McpBuild` la table et les colonnes existent en base. Après les octrois de droits de
l'étape 4, l'entité est manipulable via l'API REST dynamique (outils MCP `create_entity` /
`get_many_select`). Sans l'étape 4, `create_entity` renvoie HTTP 500 (couche 1 manquante) ou
ignore silencieusement les valeurs des champs (couche 2 manquante).

---

## 5. Récapitulatif des fichiers sources analysés

- `VPSoft.Presentation/Controllers/ApiController.cs` — route de base `api/[controller]/[action]`.
- `VPSoft.Presentation/Controllers/FormBuilderController.cs` — `AddDynamicEntity` (l.333),
  `BuildConfiguration` (l.117), `CreateDynamicField` POST (l.1356), bloc référence inverse (l.1503-1551).
- `VPSoft.Domain/Utils/Builder/Helper/BuilderConfiguration.cs` — `InheritedClassSuffix = "Extension"` (l.229).
- `VPSoft.Services/Builder/FormBuilderService.cs` — `BuildVersion` (l.77).
- `VPSoft.Services/Builder/DynamicSettingsService.cs` — `CreateDynamicSettingsCustom` (l.825),
  `IsEntityNameValid` (l.1659).
- `VPSoft.Services/Builder/ConfigurationDescriptorService.cs` — `AddPropertyWorkflow` (l.350),
  `GetOrCreateInheritedConfigurationDescriptorFromType`.
- `VPSoft.Services/Builder/DynamicFunctionService.cs` — `Create`, `EvalDynamicFunction(Async)`,
  `RunDynamicFunctionAsync`.
- `VPSoft.Services/Builder/Codes/CSharpEngineService.cs` — compilation à chaud.
- `VPSoft.Services/Helpers/Builder/Codes/DynamicCodeBuilderHelper.cs` — gabarits de génération + usings.
- `VPSoft.Presentation/Controllers/Api/VP2Controller.cs` — `POST /Api/V2/VP/Functions/Invoke`.
- `VPSoft.Utils/Extensions/VP.cs` — `VP.Functions.Invoke/InvokeAsync`.
- `VPSoft.Utils/Helpers/Process/ProcessUserService.cs` — suivi de process en mémoire.
- `VPSoft.Domain/Enums/Process/HotReloadStepState.cs` — étapes de build.
- `VPSoft.Domain/Enums/FormPropertyType.cs`, `.../Builder/ConfigurationDescriptorStatus.cs`
  (`EntityClassType`), `.../Builder/FormPropertyStatus.cs` — enums.
- `VPSoft.Domain/Contracts/DynamicModules/DynamicSettingsCreateFormModel.cs`,
  `.../DynamicFields/FormPropertyModel.cs`, `.../DynamicFields/DynamicFieldConfigFormModel.cs` — modèles.
- `VPSoft.Domain/Utils/Builder/Descriptor/FormProperty.cs` — `FormProperty`, `ReferenceRelation`,
  `ReferenceLambda` (l.236), `ValidateConstraintForCreate` (l.318).
- `VPSoft.Domain/Contracts/DynamicFields/CreateDynamicFieldFormModel.cs` — modèle d'entrée complet
  (`DynamicMultiEnum`, `ReferenceLambda`, `EntityReturnExpressionLambda`, `Category`).
- `VPSoft.Domain/Contracts/DynamicEnums/DynamicMultiEnumFormModel.cs`,
  `.../DynamicMultiEnumValuesFormModel.cs` — modèles enum.
- `VPSoft.Services.Abstractions/Builder/IDynamicMultiEnumService.cs` — `SaveOrUpdateDynamicMultiEnum` (l.22).
- `VPSoft.Services/Builder/DynamicApiService.cs` — `Post<TEntity>` (l.152, importeur), `GetUsableTable<TEntity>` (l.306-327, filtre `.Where(x => x.Api)`).
- `VPSoft.Services/Builder/DynamicFieldRoleService.cs` — `GetDynamicFieldRoleFromProperty` (l.459), `GetDynamicFieldRolesFromEntity` (l.367), `PopulateConfigToRole` (l.302, `role.Api`).
- `VPSoft.Services/Entities/Roles/RolePermissionService.cs` — `GetEntityPermissionFromAction` (l.139), Create/Edit ; lecture cache via `DynamicCacheManagerHelper.GetRolePermissions()`.
- `VPSoft.Domain/Models/Builder/DynamicFieldRole.cs` — `[EntityCache]`, `Api` (l.57), `CanRead`, `RoleActionForAdd/Edit`, `Table` ; enums `DynamicFieldRoleActionForAdd/Edit/List` (l.141-181).
- `VPSoft.Services.Abstractions/Builder/IDynamicFieldRoleService.cs`, `IDynamicFieldService.cs` — méthodes `GetDynamicFieldRoleFromProperty`, `GetDynamicFieldsFromEntity`.
- `VPSoft.Services/Builder/DynamicFieldService.cs` — `GetDynamicFieldsByAction` (l.1771) : consommateur autoritaire des filtres liste/create/edit/detail (`Table`/`RoleActionForAdd`/`RoleActionForEdit`/`CanRead`).
- `VPSoft.Services/Builder/DynamicSettingsService.cs` — `SaveDynamicSettingsConfig` (l.2805), `SaveDynamicSettingsRolesPermissions` (l.3420 : mapping `DynamicFieldRole.Table = RoleActionForTable`, etc.), `SaveDynamicSettingsProperties` (l.3265) ; cible de `/api/FormBuilder/SaveDynamicSettingsConfig`.
- `VPSoft.Domain/Contracts/Builder/Table/DynamicSettingsConfigFormModel.cs` — `DynamicSettingsRolePermissionFieldFormModel` (l.117 : `roleActionForAdd/Edit/Table`, `canRead`, `api`, …).
- `VPSoft.Services/Helpers/Builder/Compilator/CodeBuilder/Classes/ClassBuilder.cs` — génération C# du champ expression (l.153-180 : `{Name}Function(value)`, le `ReferenceLambda` doit contenir `return`), `GetRefreshExpressionFieldsMethod` (l.372-385).
- `VPSoft.Services/Helpers/Builder/Compilator/CodeBuilder/Classes/ClassMappingBuilder.cs` — mappings par catégorie : `Formula` (l.177-178), `ExpressionField` (l.181-193), `DynamicMeasures` si unité (l.293-296).
- `VPSoft.Services/Builder/DynamicFieldService.cs` — région Unit (l.3128-3161 : `FormProperty.Unit`, règle de rebuild 1ʳᵉ unité), région RenderType (l.3084-3126 : `RenderTypeOverrided`, `EntityOrViewForNestedTable`, `NestedTableSaveMode`).
- `VPSoft.Domain/Models/Builder/DynamicField.cs` — `RenderTypeOverrided` (l.1716), `FieldDisplay` (l.1263), `DateDisplay` (l.1271), `TableRenderType` (l.1313), `EntityOrViewForNestedTable` (l.1321), `NestedTableSaveMode` (l.1329), `DontDisplayLabel` (l.1833) ; enums `RenderType`, `TableRenderType` (l.2546), `NestedTableSaveMode` (l.2561).
- `VPSoft.Domain/Models/Builder/DynamicFieldName.cs` (labels : `Name`/`ColumnName`/`Tooltip`/`Culture`), `DynamicFieldDescription.cs` (`Description`/`Culture`) ; `VPSoft.Domain/Models/Entities/Culture.cs`.
- `VPSoft.Services.Abstractions/Builder/IDynamicFieldNameService.cs` (`FindDynamicFieldName`, `SaveDynamicFieldLocalizedTexts`), `IDynamicFieldDescriptionService.cs` (`SaveDynamicFieldDescription`).
- `VPSoft.Domain/Models/Entities/Unit.cs` + `VPSoft.Services.Abstractions/Entities/IUnitService.cs`, `ICultureService.cs` ; `VPSoft.Domain/Enums/UnitType.cs`, `RenderType.cs`, `FieldDisplay.cs`, `DateDisplay.cs`.
- `VPSoft.Domain/Models/ImportExport/IETableUsable.cs` — enum `IEDynamicFieldRole` (l.510 : `NotAvailable/Both/OnlyExport`), `IEType` (concept interface/intégration).
- `VPSoft.Domain/Helpers/Extensions/EntityExtension.cs` — `IsWithUnitEntity` (l.313).
- `VPSoft.Domain/Repositories/IRepositoryManager.cs` — `DynamicFieldRepository` (l.84), `FormPropertyRepository` (l.114) — utilisés pour éditer `FormProperty.Formula/ReferenceLambda/Unit` et `DynamicField.RenderTypeOverrided`.

### Fonctions MCP créées (folder API MCP `d019db8c-…`)

> ⚠️ **Ces `Id` (GUID) sont PROPRES À LA DÉMO `demo-vesta-10-5` — NON PORTABLES.** Un `DynamicFunction.Id`
> est généré par la base : sur une autre application (ou en recréant la fonction), le GUID **diffère**.
> Ils ne servent **pas** à appeler les fonctions : on les invoque **par Name**
> (`invoke_vpsoft_function("McpXxx", [...])`). Le GUID n'est requis que comme paramètre `functionId`
> (ex. `McpUpdateFunctionCode`) → le **résoudre dynamiquement** :
> `get_many_select("DynamicFunction","Id", "Name == \"McpXxx\"")`. Ce tableau = traçabilité démo uniquement.

| Fonction | Id (démo) |
|----------|-----|
| `McpCreateReverseLink` | `88883120-460d-4e23-befd-461032b2d4a5` |
| `McpCreateFormulaField` | `d3673757-2025-4f49-8e95-da9ad01dc79e` |
| `McpSetFieldFormula` | `3d0c3cf6-dbd4-4d23-9329-fe62a5f36d44` |
| `McpSetFieldExpression` | `3befde5c-d565-429d-bcb4-20f61563cae5` |
| `McpSetFieldLabel` | `f70b8ee1-732b-4fee-8ff5-c7052c38513c` |
| `McpSetFieldDescription` | `3d2c63c6-daeb-4f46-8026-e87a61a518b2` |
| `McpSetFieldExport` | `c9615bae-3b00-499c-b2fe-41abeccb52d4` |
| `McpSetFieldRender` | `9c7c2407-572d-480e-a469-645731d73ffa` |
| `McpSetFieldNestedTable` | `679d344f-120d-4cc7-abf3-eddb5784b700` |
| `McpSetFieldUnit` | `8fe9da29-e637-413e-9efe-14ea96019173` |
| `McpAddFieldsToSection` | `139d97d1-f7e7-4b4b-bed1-a7e255625d7d` |
| `McpSetFieldTexts` | `0210224e-9480-4921-9601-caba014da648` |
| `McpSetReference` | `5337c238-c87f-4c72-b8f5-5869e5f7f133` |
| `McpAddTableToMenu` | `414f4b8b-7f68-4763-9327-51459150f6ac` |
| `McpSetFieldReadOnly` | `400a01fd-5b0e-4961-94e7-1a99a7e7996c` |
| `McpSetExpressionTriggers` | `71f155ab-5ec1-4942-8dc2-5d82107487bf` |
| `McpCreateUnit` | `92f9ae07-d854-4043-9a41-570776dffcd5` |
| `McpSetWorkflowActive` | `c8dbe5d6-ae37-4cf1-b135-331f2ab561f2` |
| `McpSetFieldsImportExport` | `5658dd4f-af67-418f-840c-522878387bf8` |
| `McpSetFieldsEditInLine` | `919fbbfc-516b-4453-b977-acd68d8f1cc7` |
| `McpClearFieldUnit` | `c41e20fe-18c7-46a9-825b-52e04318f8d2` |
| `McpDeleteEntityByCode` | `de63db0b-34d5-44d1-996c-f30e21dcb97b` |
| `McpDiagMenuModel` | `f3b32e5e-d44e-4b85-83e4-785b940124ce` |
| `McpSetMenuRoles` | `c76cf51e-9e6b-49c6-b0a8-1b124d7506dc` |
| `McpGrantTablePermissionForRole` | `09d67503-4151-4adb-87a6-f6e98b6864aa` |
| `McpSetFieldsUiVisibilityForRole` | `af96c4f6-053a-4bfa-9835-d024a880a8d1` |
| `McpSetEntityCode` | `90804065-154d-4f24-90c9-ae00d86285e3` |
| `McpSetWorkflowActionCode` | `e26d282b-f6dc-40eb-a4d5-2bf76392a3e2` |
