# Bootstrap & autonomie de la skill `vpsoft-config`

Ce skill est **autoportant** : tout ce dont il a besoin est dans le `.zip` (SKILL.md + ce dossier
`references/`). Aucune dépendance au dépôt source à l'exécution.

**Seule dépendance d'exécution** : les ~50 `DynamicFunction` `Mcp*` doivent exister dans la base
de l'application VPSoft ciblée (elles portent le code C# compilé à chaud appelé via le MCP). Sur un
environnement **déjà préparé** (ex. démo `demo-vesta-10-5`), elles existent → rien à faire. Sur un
**nouvel environnement**, il faut les (re)créer. Ce fichier explique comment, à partir de la skill seule.

## Où est le code de chaque fonction
- **FormBuilder** (tables, champs, droits, visibilité, sections, rendu, unités, expression/formule,
  reverse-link, données, références) → `references/MCP-DynamicFunctions-FormBuilder.md` (code + preuves).
- **Indicateurs / dashboards / workflow** → `references/MCP-Indicators-Dashboards.md` (code + IDs).
- **Pages dynamiques** (`McpCreateDynamicPage`, `McpRenderDynamicPage`, `McpAttachFieldPage`) →
  `references/CODE-DynamicPages-VPFramework.md` §5.5 (code complet + tests ✅).
- **Boutons** (`McpCreateDynamicButton`, `McpSetButtonCode`, `McpDeleteDynamicButton`) →
  `references/CODE-DynamicPages-VPFramework.md` §9.3 ; **batchs** (`McpCreateBatch`, `McpRunBatch`) → §10.1.
- **Niveaux d'arbre** (`McpSetTreeLevels`) → `references/MCP-DynamicFunctions-FormBuilder.md` §3.1bis.
- **Business rules no-code** (`McpCreateBusinessRule` tous kinds, `McpSetBusinessRuleExtras`,
  `McpGetRuleTree`, `McpDeleteRuleTree`, `McpDeleteBusinessRule`) →
  `references/NOCODE-Conditionality-BusinessRules.md` §7 (code complet + roundtrips ✅).
- **Visuels / confidentialité / PDF / alertes / aides** (`McpAddTableWidget`, `McpDeleteTableWidget`,
  `McpSetTableConfidentiality`, `McpSetRecordConfidentiality`, `McpCreatePdfModel`, `McpCreateDataAlert`,
  `McpCreateOnlineHelp`) → `references/CONFIG-TableExtras.md` (code complet + tests ✅).
- **Workflow transitions** (`McpCreateWorkflowAction`, `McpRunWorkflowAction`, `McpDeleteWorkflowAction`)
  → `references/MCP-Indicators-Dashboards.md` §8 (code complet + roundtrip ✅).
- **Le seed `McpCreateFunction` + les helpers récents** → **ci-dessous** dans ce fichier.

## Procédure de (re)création sur un nouvel environnement
1. **Pré-requis** : le `folderId` du dossier **« API MCP »** (module `System` = Administration) où
   **doivent vivre TOUTES les `Mcp*`** (🔴 règle utilisateur impérative). Le résoudre via
   `get_many_select("DynamicFunction","DynamicFolder.Id","Name == \"McpCreateFunction\"")`, ou en UI
   (Administration → API MCP). **Toujours passer ce `folderId`** à `McpCreateFunction` (prioritaire sur
   `moduleId`) ; ne PAS passer un `moduleId` métier seul (range la fonction dans la racine du module →
   à corriger via `McpMoveFunctionsToFolder`).
2. **Créer le SEED `McpCreateFunction` MANUELLEMENT** (chicken-and-egg : on ne peut pas la créer via
   elle-même). Dans l'AGL VPSoft → éditeur de **DynamicFunction** (ou `/Builder`), créer une fonction
   nommée `McpCreateFunction` avec les `Parameters` / `CodeUsing` / `CodeFunction` ci-dessous, puis
   sauvegarder (compilation à chaud).
3. **Créer toutes les autres** via le MCP en appelant `McpCreateFunction(name, parameters, returnValue,
   codeUsing, codeFunction, isAsync, moduleId, folderId)` avec le code repris des fichiers de référence
   (FormBuilder / Indicators) et des helpers ci-dessous. `McpCreateFunction` est **idempotent sur le nom**
   (refuse un doublon) → réexécution sûre.
4. **Vérifier** : `get_many_select("DynamicFunction","Name","DynamicFolder.Id == \"<folderId>\"")`
   doit lister le catalogue (cf. SKILL.md §3).

> Rappels transverses (cf. SKILL.md) : params **tous `string`** ; **try/catch interne** obligatoire ;
> `IServiceManager`/`IRepositoryManager` via `AppDependencyResolver` ; `select_app` à **chaque tour** ;
> un `using` inexistant ⇒ échec de compilation ⇒ **retour `null` silencieux**.

---

## SEED — `McpCreateFunction` (à créer manuellement en premier)

- **Parameters** : `string name, string parameters, string returnValue, string codeUsing, string codeFunction, string isAsync, string moduleId, string folderId`
- **ReturnValue** : `System.Object` · **IsAsync** : `false`
- **CodeUsing** : `using VPSoft.Domain.Enums;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();

if (string.IsNullOrWhiteSpace(name))
    return new { success = false, message = "name requis" };

// Unicite du nom (meme controle que DynamicFunctionController.Create)
if (sm.DynamicFunctionService.GetCountByFilter(x => x.Name == name) > 0)
    return new { success = false, message = "Une fonction nommee '" + name + "' existe deja." };

// Resolution du dossier cible (folderId prioritaire, sinon racine du module)
DynamicFolder folder = null;
if (!string.IsNullOrEmpty(folderId))
    folder = sm.DynamicFolderService.GetSingle(x => x.Id == Guid.Parse(folderId));
else if (!string.IsNullOrEmpty(moduleId))
    folder = sm.DynamicFolderService.GetSingle(x => x.ParentFolder == null && x.DynamicModule.Id == Guid.Parse(moduleId));

if (folder == null)
    return new { success = false, message = "Dossier cible introuvable (fournir folderId ou moduleId valide)." };

var fn = new DynamicFunction();
fn.Name = name;
fn.Parameters = parameters;
fn.ReturnValue = string.IsNullOrEmpty(returnValue) ? "System.Object" : returnValue;
fn.CodeUsing = codeUsing;
fn.CodeFunction = codeFunction;
fn.IsAsync = (isAsync == "true" || isAsync == "1" || isAsync == "True");
fn.EntityState = EntityState.Active;
fn.Code = Guid.NewGuid().ToString();
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

---

## Helpers récents (code complet — non repris dans les autres docs)

### `McpAddTableToMenu`
- **Parameters** : `string entityName, string moduleId, string order`
- **CodeUsing** : `using VPSoft.Domain.Models.Builder;` · `using VPSoft.Domain.Models.Entities.Roles;` · `using VPSoft.Domain.Contracts.NavMenuConfig;` · `using VPSoft.Domain.Repositories;` · `using VPSoft.Utils.Helpers;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    if (string.IsNullOrWhiteSpace(moduleId)) return new { success = false, step = "args", message = "moduleId requis" };
    Guid moduleGuid = Guid.Parse(moduleId);
    int orderInt = string.IsNullOrWhiteSpace(order) ? 99 : int.Parse(order);
    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    if (dsId == Guid.Empty) return new { success = false, step = "ds", message = "DynamicSettings introuvable : " + entityName };
    var crtUser = sm.UserService.GetCurrent();
    var role = DynamicHelper.GetRoleInModuleFromModule(crtUser, moduleGuid);
    if (role == null) return new { success = false, step = "role", message = "RoleInModule introuvable pour module " + moduleId };
    var form = new NavMenuCreateOrActivateFormModel { Name = entityName, EntityId = dsId, DynamicModuleId = moduleGuid, Type = MenuType.EntityDynamic, IsForSettings = false, Order = orderInt };
    sm.NavMenuService.ActivateOrCreate(form);
    var navMenu = rm.NavMenuRepository.GetSingleFetched(x => ((NavMenuDynamicSettings)x).DynamicSettings.Id == dsId);
    if (navMenu == null) return new { success = false, step = "reload", message = "NavMenu non recharge apres ActivateOrCreate" };
    navMenu.RolePermissions = new HashSet<RoleInModule> { role };
    rm.NavMenuRepository.Edit(navMenu);
    return new { success = true, navMenuId = navMenu.Id, entityName = entityName, moduleId = moduleId, roleId = role.Id, order = orderInt, message = "Table ajoutee au menu (NavMenuDynamicSettings) + visible pour le role." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpSetFieldReadOnly`
- **Parameters** : `string entityName, string moduleId, string fieldsCsv`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Utils.Helpers;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    var whitelist = (fieldsCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    if (whitelist.Count == 0) return new { success = false, step = "args", message = "fieldsCsv requis" };
    var crtUser = sm.UserService.GetCurrent();
    var role = sm.RoleInModuleService.GetUserRoleInModuleFromEntity(crtUser, entityName);
    if (role == null && !string.IsNullOrWhiteSpace(moduleId)) role = DynamicHelper.GetRoleInModuleFromModule(crtUser, Guid.Parse(moduleId));
    if (role == null) return new { success = false, step = "role", message = "Aucun RoleInModule resolu" };
    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    if (dsId == Guid.Empty) return new { success = false, step = "ds", message = "DynamicSettings introuvable" };
    var ds = sm.DynamicSettingsService.GetSingle(dsId);
    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();
    var applied = new List<string>();
    foreach (var field in fields)
    {
        if (!whitelist.Contains(field.PropertyName)) continue;
        var dfr = sm.DynamicFieldRoleService.GetDynamicFieldRoleFromProperty(field, null, role.Id, false);
        if (dfr == null)
        {
            dfr = new DynamicFieldRole { EntityState = EntityState.Active, DynamicField = field, RoleInModule = role, DynamicSettings = ds, CanRead = true, Table = DynamicFieldRoleActionForList.Display, RoleActionForAdd = DynamicFieldRoleActionForAdd.ReadOnly, RoleActionForEdit = DynamicFieldRoleActionForEdit.ReadOnly };
            sm.DynamicFieldRoleService.Create(dfr);
        }
        else
        {
            dfr.RoleActionForAdd = DynamicFieldRoleActionForAdd.ReadOnly;
            dfr.RoleActionForEdit = DynamicFieldRoleActionForEdit.ReadOnly;
            dfr.CanRead = true;
            sm.DynamicFieldRoleService.Edit(dfr);
        }
        applied.Add(field.PropertyName);
    }
    return new { success = true, entityName = entityName, roleId = role.Id, readOnly = applied, message = "Champs en lecture seule (Add/Edit=ReadOnly), Table conservee." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpSetExpressionTriggers`
- **Parameters** : `string entityName, string expressionFieldName, string triggerFieldNamesCsv`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Repositories;`

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    if (string.IsNullOrWhiteSpace(expressionFieldName)) return new { success = false, step = "args", message = "expressionFieldName requis" };
    var names = (triggerFieldNamesCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    if (names.Count == 0) return new { success = false, step = "args", message = "triggerFieldNamesCsv requis" };
    var field = rm.DynamicFieldRepository.GetSingle(x => x.EntityName == entityName && x.PropertyName == expressionFieldName && x.ViewName == null, x => x.ExpressionTriggerFields);
    if (field == null) return new { success = false, step = "field", message = "Champ expression introuvable : " + entityName + "." + expressionFieldName };
    var all = rm.DynamicFieldRepository.GetMany(x => x.EntityName == entityName && x.ViewName == null).ToList();
    var triggers = all.Where(f => names.Contains(f.PropertyName)).ToHashSet();
    if (triggers.Count == 0) return new { success = false, step = "triggers", message = "Aucun champ trigger trouve parmi : " + triggerFieldNamesCsv };
    field.ExpressionTriggerFields = triggers;
    rm.DynamicFieldRepository.Edit(field);
    return new { success = true, entityName = entityName, expressionField = expressionFieldName, triggers = triggers.Select(t => t.PropertyName).ToList(), message = "Trigger fields poses (MAJ live). Pas de rebuild requis." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpCreateUnit`
- **Parameters** : `string name, string code, string unitTypeString, string baseMeasureSymbol, string baseMeasureCode, string description`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Contracts.Units;` · `using VPSoft.Domain.Models.Entities;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(name) || string.IsNullOrWhiteSpace(code) || string.IsNullOrWhiteSpace(unitTypeString))
        return new { success = false, step = "args", message = "name, code, unitTypeString requis" };
    var form = new UnitCreateFormModel { Name = name, Code = code, Description = description, UnitTypeString = unitTypeString };
    string error;
    bool ok = sm.UnitService.CreateUnit(form, out error);
    if (!ok) return new { success = false, step = "createUnit", message = string.IsNullOrEmpty(error) ? "Echec CreateUnit" : error };
    var unit = sm.UnitService.GetSingle(x => x.Code == code);
    if (unit == null) return new { success = false, step = "reload", message = "Unit non rechargee apres creation" };
    Guid? measureId = null;
    if (!string.IsNullOrWhiteSpace(baseMeasureSymbol))
    {
        var measure = new Measure { Symbol = baseMeasureSymbol, Code = string.IsNullOrWhiteSpace(baseMeasureCode) ? baseMeasureSymbol : baseMeasureCode, OrderBy = 0, Description = baseMeasureSymbol, Value = 1, IsBasicUnit = true, Unit = unit, UnitType = unit.UnitType };
        if (sm.MeasureService.Create(measure)) measureId = measure.Id;
    }
    return new { success = true, unitId = unit.Id, unitCode = unit.Code, unitType = unit.UnitType.ToString(), measureId = measureId, baseMeasure = baseMeasureSymbol, message = "Unite + mesure de base creees." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpSetWorkflowActive`
- **Parameters** : `string entityName, string active`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Models.Workflow;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    bool act = (active == "true" || active == "1" || active == "True");
    var wfs = sm.EntityWorkflowService.FindAll(x => x.EntityName == entityName).ToList();
    if (!wfs.Any()) return new { success = false, step = "find", message = "Aucun EntityWorkflow pour " + entityName };
    var changed = new List<object>();
    foreach (var wf in wfs)
    {
        wf.EntityState = act ? EntityState.Active : EntityState.Inactive;
        sm.EntityWorkflowService.Edit(wf);
        changed.Add(new { id = wf.Id, name = wf.Name, state = wf.EntityState.ToString() });
    }
    return new { success = true, entityName = entityName, active = act, workflows = changed, message = act ? "Workflow(s) active(s)." : "Workflow(s) desactive(s) : masque(s) des formulaires, rien supprime (reversible)." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpSetFieldsImportExport`
- **Parameters** : `string entityName, string moduleId, string mode, string fieldsCsv` (`mode` = `Both`/`OnlyExport`/`NotAvailable`)
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Utils.Helpers;` · `using VPSoft.Domain.Models.ImportExport;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    var list = (fieldsCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    if (list.Count == 0) return new { success = false, step = "args", message = "fieldsCsv requis" };
    IEDynamicFieldRole target;
    if (!Enum.TryParse<IEDynamicFieldRole>(mode, true, out target)) return new { success = false, step = "mode", message = "mode invalide (Both/OnlyExport/NotAvailable)" };
    var crtUser = sm.UserService.GetCurrent();
    var role = sm.RoleInModuleService.GetUserRoleInModuleFromEntity(crtUser, entityName);
    if (role == null && !string.IsNullOrWhiteSpace(moduleId)) role = DynamicHelper.GetRoleInModuleFromModule(crtUser, Guid.Parse(moduleId));
    if (role == null) return new { success = false, step = "role", message = "Aucun RoleInModule resolu" };
    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    var ds = sm.DynamicSettingsService.GetSingle(dsId);
    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();
    var applied = new List<string>();
    foreach (var field in fields)
    {
        if (!list.Contains(field.PropertyName)) continue;
        var dfr = sm.DynamicFieldRoleService.GetDynamicFieldRoleFromProperty(field, null, role.Id, false);
        if (dfr == null) { dfr = new DynamicFieldRole { EntityState = EntityState.Active, DynamicField = field, RoleInModule = role, DynamicSettings = ds, CanRead = true, ImportExport = target }; sm.DynamicFieldRoleService.Create(dfr); }
        else { dfr.ImportExport = target; sm.DynamicFieldRoleService.Edit(dfr); }
        applied.Add(field.PropertyName);
    }
    return new { success = true, entityName = entityName, mode = target.ToString(), applied = applied };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpSetFieldsEditInLine`
- **Parameters** : `string entityName, string moduleId, string fieldsCsv, string enable`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Utils.Helpers;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    var list = (fieldsCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    if (list.Count == 0) return new { success = false, step = "args", message = "fieldsCsv requis" };
    bool en = (enable == "true" || enable == "1" || enable == "True");
    var crtUser = sm.UserService.GetCurrent();
    var role = sm.RoleInModuleService.GetUserRoleInModuleFromEntity(crtUser, entityName);
    if (role == null && !string.IsNullOrWhiteSpace(moduleId)) role = DynamicHelper.GetRoleInModuleFromModule(crtUser, Guid.Parse(moduleId));
    if (role == null) return new { success = false, step = "role", message = "Aucun RoleInModule resolu" };
    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    var ds = sm.DynamicSettingsService.GetSingle(dsId);
    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();
    var applied = new List<string>();
    var notFound = new List<string>(list);
    foreach (var field in fields)
    {
        if (!list.Contains(field.PropertyName)) continue;
        notFound.Remove(field.PropertyName);
        var dfr = sm.DynamicFieldRoleService.GetDynamicFieldRoleFromProperty(field, null, role.Id, false);
        if (dfr == null) { dfr = new DynamicFieldRole { EntityState = EntityState.Active, DynamicField = field, RoleInModule = role, DynamicSettings = ds, CanRead = true, Table = DynamicFieldRoleActionForList.Display, EditInLine = en }; sm.DynamicFieldRoleService.Create(dfr); }
        else { dfr.EditInLine = en; sm.DynamicFieldRoleService.Edit(dfr); }
        applied.Add(field.PropertyName);
    }
    return new { success = true, entityName = entityName, editInLine = en, applied = applied, notFound = notFound, message = "EditInLine mis a jour." };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

### `McpClearFieldUnit`
- **Parameters** : `string entityName, string fieldName`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Domain.Utils.Builder.Descriptor;` · `using VPSoft.Domain.Repositories;`

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try
{
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    if (string.IsNullOrWhiteSpace(fieldName)) return new { success = false, step = "args", message = "fieldName requis" };
    var field = rm.DynamicFieldRepository.GetSingle(x => x.EntityName == entityName && x.PropertyName == fieldName && x.ViewName == null);
    if (field == null) return new { success = false, step = "field", message = "Champ introuvable : " + entityName + "." + fieldName };
    var fp = field.FormProperty;
    if (fp == null) return new { success = false, step = "formProperty", message = "FormProperty null" };
    bool had = fp.Unit != null;
    fp.Unit = null;
    fp.Status = FormPropertyStatus.MODIFIED;
    field.Status = FormPropertyStatus.MODIFIED;
    rm.DynamicFieldRepository.Edit(field);
    rm.FormPropertyRepository.Edit(fp);
    return new { success = true, entityName = entityName, fieldName = fieldName, hadUnit = had, message = "Unite retiree. Lancer McpBuild pour restaurer l'ecriture REST." };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpDeleteEntityByCode`
- **Parameters** : `string entityName, string code` (`code` = la valeur du champ entity-code `Code`, pas un champ métier)
- **CodeUsing** : `using VPSoft.Domain.Helpers;` · `using VPSoft.Domain.Helpers.Extensions;` · `using NHibernate;` · `using NHibernate.Criterion;`

```csharp
var session = NHSessionHelper.GetCurrentSession();
try
{
    if (string.IsNullOrWhiteSpace(entityName) || string.IsNullOrWhiteSpace(code))
        return new { success = false, step = "args", message = "entityName et code requis" };
    Type t = ReflectionHelper.GetTypeEntity(entityName);
    if (t == null) return new { success = false, step = "type", message = "Entite introuvable : " + entityName };
    var e = session.CreateCriteria(t).Add(Restrictions.Eq("Code", code)).UniqueResult();
    if (e == null) return new { success = false, step = "find", message = "Introuvable (Code=" + code + ")" };
    session.Delete(e);
    session.Flush();
    return new { success = true, entityName = entityName, code = code, message = "Enregistrement supprime." };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpSetDashboardOwner`
- **Parameters** : `string dashboardId, string userId` (transfert de propriété d'un dashboard → débloque l'édition, cf. SKILL.md §9 ; l'audit ne pose `CreatedUser` qu'au `Save`, pas à l'`Edit` → réaffectation OK)
- **CodeUsing** : `using VPSoft.Domain.Repositories;` · `using VPSoft.Domain.Utils.Dependency;`

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try
{
    var dId = Guid.Parse(dashboardId);
    var uId = Guid.Parse(userId);
    var dash = rm.DashboardRepository.GetSingle(x => x.Id == dId);
    if (dash == null) return new { success = false, message = "dashboard introuvable: " + dashboardId };
    var oldOwner = dash.CreatedUser == null ? "null" : dash.CreatedUser.Id.ToString();
    var user = rm.UserRepository.LoadReference(uId);
    dash.CreatedUser = user;
    rm.DashboardRepository.Edit(dash);
    return new { success = true, dashboardId = dashboardId, oldOwnerId = oldOwner, newOwnerId = userId };
}
catch (Exception ex)
{
    return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message };
}
```

---

## Helpers filtres / menus / pages interactives (session 2026-06) — code complet ✅

### `McpSetFieldsFilter` — colonnes filtrables (bandeau de liste)
- **Parameters** : `string entityName, string moduleId, string fieldsCsv, string enable`
- **CodeUsing** : `using VPSoft.Domain.Enums;` · `using VPSoft.Utils.Helpers;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
try {
    if (string.IsNullOrWhiteSpace(entityName)) return new { success = false, step = "args", message = "entityName requis" };
    var list = (fieldsCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    if (list.Count == 0) return new { success = false, step = "args", message = "fieldsCsv requis" };
    bool en = (enable == "true" || enable == "1" || enable == "True");
    var crtUser = sm.UserService.GetCurrent();
    var role = sm.RoleInModuleService.GetUserRoleInModuleFromEntity(crtUser, entityName);
    if (role == null && !string.IsNullOrWhiteSpace(moduleId)) role = DynamicHelper.GetRoleInModuleFromModule(crtUser, Guid.Parse(moduleId));
    if (role == null) return new { success = false, step = "role", message = "Aucun RoleInModule resolu" };
    Guid dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    var ds = sm.DynamicSettingsService.GetSingle(dsId);
    var fields = sm.DynamicFieldService.GetDynamicFieldsFromEntity(entityName, null, false, true, true).ToList();
    var applied = new List<string>(); var notFound = new List<string>(list);
    foreach (var field in fields) {
        if (!list.Contains(field.PropertyName)) continue;
        notFound.Remove(field.PropertyName);
        var dfr = sm.DynamicFieldRoleService.GetDynamicFieldRoleFromProperty(field, null, role.Id, false);
        if (dfr == null) { dfr = new DynamicFieldRole { EntityState = EntityState.Active, DynamicField = field, RoleInModule = role, DynamicSettings = ds, CanRead = true, Table = DynamicFieldRoleActionForList.Display, Filter = en }; sm.DynamicFieldRoleService.Create(dfr); }
        else { dfr.Filter = en; sm.DynamicFieldRoleService.Edit(dfr); }
        applied.Add(field.PropertyName);
    }
    return new { success = true, entityName = entityName, filter = en, applied = applied, notFound = notFound };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpSetNavMenuIconOrderBulk` — icône + ordre des entrées de menu
- **Parameters** : `string itemsJson` (= `[{"id":"<navMenuId>","icon":"mdi-…","order":N}, …]`, clé absente = inchangée)
- **CodeUsing** : `using VPSoft.Domain.Repositories;`

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try {
    if (string.IsNullOrWhiteSpace(itemsJson)) return new { success = false, step = "args", message = "itemsJson requis" };
    var arr = Newtonsoft.Json.Linq.JArray.Parse(itemsJson);
    var done = new List<object>();
    foreach (var it in arr) {
        var idStr = (string)it["id"]; if (string.IsNullOrWhiteSpace(idStr)) { done.Add(new { ok = false, msg = "id manquant" }); continue; }
        var id = Guid.Parse(idStr);
        var nm = rm.NavMenuRepository.GetSingle(x => x.Id == id);
        if (nm == null) { done.Add(new { id = idStr, ok = false, msg = "NavMenu introuvable" }); continue; }
        if (it["icon"] != null) nm.Icon = (string)it["icon"];
        if (it["order"] != null) nm.OrderBy = (int)it["order"];
        rm.NavMenuRepository.Edit(nm);
        done.Add(new { id = idStr, ok = true, icon = nm.Icon, order = nm.OrderBy });
    }
    return new { success = true, items = done };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpAddPageToMenu` — ajouter une DynamicPage au menu d'un module
- **Parameters** : `string pageCode, string moduleId, string name, string label, string icon, string order, string roleCodesCsv`
- **CodeUsing** : `using VPSoft.Domain.Models.Builder;` · `using VPSoft.Domain.Models.Entities.Roles;` · `using VPSoft.Domain.Contracts.NavMenuConfig;` · `using VPSoft.Domain.Repositories;` · `using VPSoft.Utils.Helpers;`

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try {
    var page = rm.DynamicPageBaseRepository.GetMany(x => x.Code == pageCode).FirstOrDefault();
    if (page == null) return new { success = false, message = "page introuvable: " + pageCode };
    Guid moduleGuid = Guid.Parse(moduleId);
    int orderInt = string.IsNullOrWhiteSpace(order) ? 1 : int.Parse(order);
    var form = new NavMenuCreateOrActivateFormModel { Name = name, EntityId = page.Id, DynamicModuleId = moduleGuid, Type = MenuType.DynamicPage, IsForSettings = false, Order = orderInt };
    sm.NavMenuService.ActivateOrCreate(form);
    var navMenu = rm.NavMenuRepository.GetSingleFetched(x => ((NavMenuDynamicPage)x).DynamicPage.Id == page.Id);
    if (navMenu == null) return new { success = false, message = "navMenu non recharge" };
    if (!string.IsNullOrEmpty(icon)) navMenu.Icon = icon;
    navMenu.OrderBy = orderInt; navMenu.IsActiveUser = true;
    var codes = (roleCodesCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s != "").ToList();
    var roles = rm.RoleInModuleRepository.GetMany(x => codes.Contains(x.Code) && x.DynamicModule.Id == moduleGuid).ToList();
    if (roles.Count > 0) navMenu.RolePermissions = new HashSet<RoleInModule>(roles);
    rm.NavMenuRepository.Edit(navMenu);
    if (!string.IsNullOrEmpty(label)) {
        var rc = new ResourceCultureJson { resourceKey = "LocalizedName", resourceValues = new List<ResourceCultureValueJson> {
            new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = label },
            new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = label } } };
        sm.CultureParameterService.SaveOrUpdate(rc, navMenu.Id, typeof(NavMenu).GetProperty("LocalizedName"));
    }
    return new { success = true, navMenuId = navMenu.Id, roles = roles.Count, icon = navMenu.Icon, order = orderInt };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpSetDynamicPageRoles` — rôles d'accès d'une page/widget (sinon `UnauthorizedAccessException`)
- **Parameters** : `string pageCode, string roleCodesCsv` · **CodeUsing** : `using VPSoft.Domain.Repositories;` · `using VPSoft.Domain.Models.Entities.Roles;`

```csharp
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try {
    if (string.IsNullOrWhiteSpace(pageCode)) return new { success = false, message = "pageCode requis" };
    var codes = (roleCodesCsv ?? "").Split(',').Select(s => s.Trim()).Where(s => s.Length > 0).ToList();
    if (codes.Count == 0) return new { success = false, message = "roleCodesCsv requis" };
    var page = rm.DynamicPageBaseRepository.GetMany(x => x.Code == pageCode).FirstOrDefault();
    if (page == null) return new { success = false, message = "DynamicPage introuvable: " + pageCode };
    var roles = rm.RoleInModuleRepository.GetMany(x => codes.Contains(x.Code)).ToList();
    if (roles.Count == 0) return new { success = false, message = "Aucun RoleInModule pour: " + roleCodesCsv };
    page.RoleInModules = new HashSet<RoleInModule>(roles);
    rm.DynamicPageBaseRepository.Edit(page);
    return new { success = true, pageCode = pageCode, rolesSet = roles.Select(r => r.Code).Distinct().ToList(), count = roles.Count };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpUpdateDynamicPageCode` — MAJ du code d'une page (param vide = conservé)
- **Parameters** : `string code, string codeRazor, string codeCss, string codeJavascript` · **CodeUsing** : `using VPSoft.Domain.Repositories;`
- ⚠️ `DynamicPageCodeViewModel.GitFileBasePath` est `required` → **passer par `GetCode(id)`** (pas un `new {}` qui ne compile pas / renvoie null).

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try {
    if (string.IsNullOrWhiteSpace(code)) return new { success = false, message = "code requis" };
    var page = rm.DynamicPageBaseRepository.GetMany(x => x.Code == code).FirstOrDefault();
    if (page == null) return new { success = false, message = "DynamicPage introuvable: " + code };
    var vm = sm.DynamicPageBaseService.GetCode(page.Id);
    if (vm == null) return new { success = false, message = "GetCode null pour: " + code };
    if (!string.IsNullOrEmpty(codeRazor)) vm.CodeRazor = codeRazor;
    if (!string.IsNullOrEmpty(codeJavascript)) vm.CodeJavascript = codeJavascript;
    if (!string.IsNullOrEmpty(codeCss)) vm.CodeCss = codeCss;
    bool merge = sm.DynamicPageBaseService.SaveCode(vm);
    return new { success = true, code = code, id = page.Id, hasNewToMerge = merge };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpSetDynamicPageJsB64` — injecter un gros JS via base64 (contourne l'échappement)
- **Parameters** : `string code, string b64, string reset, string finalize` · **CodeUsing** : `using VPSoft.Domain.Repositories;`
- Procédure : `base64 -i fichier.js | tr -d '\n'`, découper en chunks de ~6 KB, **`cat` un chunk à la fois** (sortie = exactement le chunk → pas de débordement de frontière), envoyer : 1ᵉʳ chunk `reset="true"`, dernier `finalize="true"`. Le helper accumule le base64 puis décode. ⚠️ Toujours `node --check` le JS d'abord ; après, `McpRenderDynamicPage`. (Idem possible pour CSS/Razor en adaptant.)

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
var rm = AppDependencyResolver.GetService<IRepositoryManager>();
try {
    var page = rm.DynamicPageBaseRepository.GetMany(x => x.Code == code).FirstOrDefault();
    if (page == null) return new { success = false, message = "introuvable: " + code };
    var vm = sm.DynamicPageBaseService.GetCode(page.Id);
    if (vm == null) return new { success = false, message = "GetCode null" };
    string buf = (reset == "true" ? "" : (vm.CodeJavascript ?? "")) + (b64 ?? "");
    bool fin = finalize == "true";
    if (fin) buf = System.Text.Encoding.UTF8.GetString(Convert.FromBase64String(buf));
    vm.CodeJavascript = buf;
    sm.DynamicPageBaseService.SaveCode(vm);
    return new { success = true, code = code, finalized = fin, total = buf.Length };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName }; }
```

### `McpSetColumnWidth` — largeur Bootstrap des colonnes d'une section VISUELS (corrige le 3+1)
- **Parameters** : `string sectionId, string classWidth` (`classWidth` = `"col"` recommandé pour N cartes sur une ligne ; cf. CONFIG-TableExtras.md §1.3 gotcha) · **CodeUsing** : `using VPSoft.Domain.Helpers;` · `using NHibernate;` · `using NHibernate.Criterion;` · `using VPSoft.Domain.Models.Builder;`
- Passe **toutes** les colonnes (`DynamicColumnTemplate`) d'une section (`DynamicSectionTemplate.Id == sectionId`) au `ClassWidth` donné. NHibernate direct ; **aucun cache** sur ces entités → effet au prochain chargement de la liste (F5). Usage : `sectionId` = l'`Id` retourné par `McpAddTableWidget` (ou `get_many_select("DynamicSectionTemplate","Id","Code == \"MCP_…\"")`).

```csharp
var session = NHSessionHelper.GetCurrentSession();
try
{
    if (string.IsNullOrWhiteSpace(sectionId) || string.IsNullOrWhiteSpace(classWidth))
        return new { success = false, step = "args", message = "sectionId et classWidth requis" };
    var sid = Guid.Parse(sectionId);
    var cols = session.CreateCriteria(typeof(DynamicColumnTemplate)).CreateAlias("DynamicSectionTemplate", "s").Add(Restrictions.Eq("s.Id", sid)).List<DynamicColumnTemplate>();
    int n = 0;
    var olds = new System.Collections.Generic.List<string>();
    foreach (DynamicColumnTemplate c in cols)
    {
        olds.Add(c.ClassWidth);
        c.ClassWidth = classWidth;
        session.Update(c);
        n++;
    }
    session.Flush();
    return new { success = true, sectionId = sectionId, updated = n, newClassWidth = classWidth, oldClassWidths = olds };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpSetSectionRole` — rattacher une section VISUELS au BON `RoleInModule` (par module)
- **Parameters** : `string sectionId, string roleInModuleId` (⚠️ `roleInModuleId` = le `RoleInModule.Id` du **module de l'entité**, cf. CONFIG-TableExtras.md §1.3 piège racine) · **CodeUsing** : `using VPSoft.Domain.Helpers;` · `using VPSoft.Domain.Models.Builder;` · `using VPSoft.Domain.Models.Entities.Roles;`
- Corrige une section créée pour un rôle homonyme du **mauvais** module (visible sur la liste user mais introuvable dans l'admin Visuels). NHibernate direct ; aucun cache → effet au F5. Résoudre le bon Id : `get_many_select("RoleInModule","Id,DynamicModule.Code","Code == \"VPWAdmin\"")` → prendre la ligne du module de l'entité. (`McpAddTableWidget` est désormais corrigée pour résoudre par module → ce retrofit ne sert que pour les sections créées avant le fix.)

```csharp
var session = NHSessionHelper.GetCurrentSession();
try
{
    if (string.IsNullOrWhiteSpace(sectionId) || string.IsNullOrWhiteSpace(roleInModuleId))
        return new { success = false, step = "args", message = "sectionId et roleInModuleId requis" };
    var sid = Guid.Parse(sectionId);
    var rid = Guid.Parse(roleInModuleId);
    var section = session.Get<DynamicSectionTemplate>(sid);
    if (section == null) return new { success = false, step = "section", message = "section introuvable: " + sectionId };
    var role = session.Get<RoleInModule>(rid);
    if (role == null) return new { success = false, step = "role", message = "RoleInModule introuvable: " + roleInModuleId };
    var oldRole = section.Role == null ? "null" : section.Role.Id.ToString();
    section.Role = role;
    session.Update(section);
    session.Flush();
    return new { success = true, sectionId = sectionId, oldRoleId = oldRole, newRoleId = roleInModuleId };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

> **Correctif `McpAddTableWidget` (par module)** : la résolution du rôle doit être module-scopée —
> `var moduleCode = sm.DynamicModuleService.GetDynamicModuleCodeFromEntityName(entityName);` puis
> `rm.RoleInModuleRepository.GetMany(x => x.Code == roleCode && x.DynamicModule.Code == moduleCode).FirstOrDefault();`
> (le code complet à jour est dans `CONFIG-TableExtras.md` §1.3).

### `McpListModuleFolders` — lister les dossiers d'un module (le WHERE-on-nav échoue côté API)
- **Parameters** : `string moduleId` · **CodeUsing** : `using VPSoft.Domain.Helpers;` · `using NHibernate;` · `using NHibernate.Criterion;` · `using VPSoft.Domain.Models.Builder;`
- ⚠️ `get_many_select("DynamicFolder", … , where:"DynamicModule.Id == …")` renvoie **"SQL not available"** (la nav `DynamicFolder→DynamicModule` n'est pas filtrable côté API V2). Passer par NHibernate (Criteria + alias).

```csharp
var session = NHSessionHelper.GetCurrentSession();
try
{
    var mid = Guid.Parse(moduleId);
    var folders = session.CreateCriteria(typeof(DynamicFolder)).CreateAlias("DynamicModule", "m").Add(Restrictions.Eq("m.Id", mid)).List<DynamicFolder>();
    var res = new System.Collections.Generic.List<object>();
    foreach (DynamicFolder f in folders) res.Add(new { id = f.Id, code = f.Code, name = f.LocalizedName, parent = f.ParentFolder == null ? "ROOT" : f.ParentFolder.Id.ToString() });
    return new { success = true, count = res.Count, folders = res };
}
catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpCreateFolderMovePages` — créer un dossier nommé dans un module + y ranger des pages/widgets
- **Parameters** : `string parentFolderId, string folderCode, string folderName, string pageCodesCsv` · **CodeUsing** : `using VPSoft.Domain.Helpers;` · `using NHibernate;` · `using VPSoft.Domain.Models.Builder;` · `using VPSoft.Domain.Models.Entities;` · `using VPSoft.Domain.Contracts.App;`
- ⚠️ **Les pages/widgets créés via MCP atterrissent dans le dossier `API MCP` du module `System`** (héritage du dossier de `McpCreateFunction`) → pour qu'un consultant les retrouve, les **ranger dans un dossier du bon module**. Reproduit le chemin officiel `DynamicFolderController.Create` (`:49-62`) : crée le `DynamicFolder` (Code + ParentFolder + DynamicModule hérité du parent) puis pose le **nom localisé** via `CultureParameterService.SaveOrUpdate(ResourceCultureJson, folder.Id, typeof(DynamicFolder).GetProperty("LocalizedName"))` pour **toutes les cultures**. Sans CultureParameter, `DynamicFolder.LocalizedName` retombe sur `Code` (`DynamicFolder.cs:42`). `parentFolderId` = la racine du module (`McpListModuleFolders` → la ligne `parent:"ROOT"`).

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    var session = NHSessionHelper.GetCurrentSession();
    if (string.IsNullOrWhiteSpace(parentFolderId) || string.IsNullOrWhiteSpace(folderCode)) return new { success = false, message = "parentFolderId et folderCode requis" };
    var parent = sm.DynamicFolderService.GetSingle(Guid.Parse(parentFolderId));
    if (parent == null) return new { success = false, message = "parent folder introuvable" };
    if (sm.DynamicFolderService.GetCountByFilter(x => x.Code == folderCode) > 0) return new { success = false, message = "Code dossier deja utilise: " + folderCode };
    var folder = new DynamicFolder();
    folder.ParentFolder = parent;
    folder.DynamicModule = parent.DynamicModule;
    folder.Code = folderCode;
    if (!sm.DynamicFolderService.Create(folder)) return new { success = false, message = "echec creation dossier" };
    var rcj = new ResourceCultureJson();
    rcj.resourceKey = "LocalizedName"; rcj.propertyName = "LocalizedName";
    rcj.resourceValues = new System.Collections.Generic.List<ResourceCultureValueJson>();
    foreach (Culture c in session.CreateCriteria(typeof(Culture)).List<Culture>()) rcj.resourceValues.Add(new ResourceCultureValueJson { cultureCode = c.CultureCode, resourceValue = folderName });
    sm.CultureParameterService.SaveOrUpdate(rcj, folder.Id, typeof(DynamicFolder).GetProperty("LocalizedName"));
    var moved = new System.Collections.Generic.List<string>();
    foreach (var raw in pageCodesCsv.Split(',')) {
        var code = raw.Trim(); if (code == "") continue;
        var page = rm.DynamicPageBaseRepository.GetMany(x => x.Code == code).FirstOrDefault();
        if (page == null) { moved.Add(code + ":NOTFOUND"); continue; }
        page.DynamicFolder = folder; rm.DynamicPageBaseRepository.Edit(page); moved.Add(code + ":OK");
    }
    session.Flush();
    return new { success = true, folderId = folder.Id, folderCode = folder.Code, folderName = folderName, moved = moved };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `McpMoveFunctionsToFolder` — ranger des `Mcp*` dans le bon dossier (« API MCP »)
- **Parameters** : `string functionNamesCsv, string folderId` · **CodeUsing** : `using VPSoft.Domain.Helpers;` · `using NHibernate;` · `using NHibernate.Criterion;` · `using VPSoft.Domain.Models.Builder;`
- 🔴 **RÈGLE UTILISATEUR (impérative)** : **toutes les fonctions `Mcp*` doivent vivre dans le dossier « API MCP » (module `System` = Administration)**, jamais dans un module métier. À la création (`McpCreateFunction`), **passer `folderId` = l'Id « API MCP »** (prioritaire sur `moduleId`). Cette fonction **rapatrie** les fonctions mal rangées. Résoudre l'Id « API MCP » par base : `get_many_select("DynamicFunction","DynamicFolder.Id","Name == \"McpCreateFunction\"")` (démo : `d019db8c-8b9a-4922-b3d2-d3145638b1b2`). Vérif : `get_many_select("DynamicFunction","Name,DynamicFolder.LocalizedName","Name.StartsWith(\"Mcp\")")` → toutes en « API MCP ».

```csharp
var session = NHSessionHelper.GetCurrentSession();
try {
    if (string.IsNullOrWhiteSpace(functionNamesCsv) || string.IsNullOrWhiteSpace(folderId)) return new { success = false, message = "functionNamesCsv et folderId requis" };
    var folder = session.Get<DynamicFolder>(Guid.Parse(folderId));
    if (folder == null) return new { success = false, message = "folder introuvable: " + folderId };
    var moved = new System.Collections.Generic.List<string>();
    foreach (var raw in functionNamesCsv.Split(',')) {
        var name = raw.Trim(); if (name == "") continue;
        var fn = (DynamicFunction)session.CreateCriteria(typeof(DynamicFunction)).Add(Restrictions.Eq("Name", name)).UniqueResult();
        if (fn == null) { moved.Add(name + ":NOTFOUND"); continue; }
        var old = fn.DynamicFolder == null ? "null" : fn.DynamicFolder.Id.ToString();
        fn.DynamicFolder = folder; session.Update(fn); moved.Add(name + ":OK(was " + old + ")");
    }
    session.Flush();
    return new { success = true, folderId = folderId, folderName = folder.LocalizedName, moved = moved };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

### `BdgCreate` — créer une entité dynamique AVEC références (générique)
- **Parameters** : `string entityName, string fieldsJson` (objet JSON `{champ:valeur}`) · **CodeUsing** : `using NHibernate;` · `using NHibernate.Criterion;` · `using VPSoft.Domain.Helpers;` · `using VPSoft.Domain.Helpers.Extensions;`
- ⚠️ À appeler côté page via `VP.Functions.Invoke("BdgCreate",[entityName, JSON.stringify(data)])`. `VP.Entities.Create` (endpoint V2) ne lie PAS les références (cf. CODE-DynamicPages §13). Réf = champ chargé par `Code` ; enum = int ; Code auto si absent ; audit posé. (Nom « Bdg » historique — générique : renommer en `McpCreateEntity` au besoin.)

```csharp
var sm = AppDependencyResolver.GetService<IServiceManager>();
var session = NHSessionHelper.GetCurrentSession();
try {
    var t = ReflectionHelper.GetTypeEntity(entityName);
    if (t == null) return new { success = false, message = "entite introuvable: " + entityName };
    var obj = Activator.CreateInstance(t);
    var data = Newtonsoft.Json.Linq.JObject.Parse(fieldsJson);
    var setList = new List<string>();
    foreach (var p in data.Properties()) {
        var prop = t.GetProperty(p.Name); if (prop == null || !prop.CanWrite) continue;
        var val = p.Value; if (val.Type == Newtonsoft.Json.Linq.JTokenType.Null) continue;
        Type pt = prop.PropertyType; Type nt = Nullable.GetUnderlyingType(pt) ?? pt; object toSet = null;
        if (nt.IsEnum) toSet = Enum.ToObject(nt, (int)val);
        else if (nt == typeof(decimal)) toSet = (decimal)val;
        else if (nt == typeof(int)) toSet = (int)val;
        else if (nt == typeof(double)) toSet = (double)val;
        else if (nt == typeof(bool)) toSet = (bool)val;
        else if (nt == typeof(DateTime)) toSet = (DateTime)val;
        else if (nt == typeof(string)) toSet = (string)val;
        else if (nt.IsClass && nt.GetProperty("Id") != null && nt.GetProperty("Id").PropertyType == typeof(Guid)) {
            var code = (string)val; if (string.IsNullOrEmpty(code)) continue;
            toSet = session.CreateCriteria(nt).Add(Restrictions.Eq("Code", code)).UniqueResult();
            if (toSet == null) { setList.Add(p.Name + "=REF_INTROUVABLE(" + code + ")"); continue; }
        } else continue;
        prop.SetValue(obj, toSet); setList.Add(p.Name);
    }
    var codeProp = t.GetProperty("Code");
    if (codeProp != null) { var cur = codeProp.GetValue(obj) as string; if (string.IsNullOrEmpty(cur)) codeProp.SetValue(obj, (entityName.Length >= 3 ? entityName.Substring(0, 3).ToUpper() : entityName) + "-" + Guid.NewGuid().ToString("N").Substring(0, 10).ToUpper()); }
    var cu = sm.UserService.GetCurrent();
    var cuProp = t.GetProperty("CreatedUser"); if (cuProp != null && cu != null) cuProp.SetValue(obj, cu);
    var cdProp = t.GetProperty("CreatedDate"); if (cdProp != null) cdProp.SetValue(obj, DateTime.Now);
    session.Save(obj); session.Flush();
    return new { success = true, entityName = entityName, code = codeProp != null ? codeProp.GetValue(obj) as string : null, set = setList };
} catch (Exception ex) { return new { success = false, step = "exception", message = ex.Message, type = ex.GetType().FullName, inner = ex.InnerException == null ? null : ex.InnerException.Message }; }
```

> **Correctif Boolean** : dans `McpCreateField`, la branche `if (fp.Type == FormPropertyType.Boolean)` doit
> aussi poser **`fp.DefaultValue = "0";`** (en plus de `Nullable=false; HasDefaultValue=true;`) — sinon la
> création échoue (`"DefaultValue doit être 0 ou 1"`). Le code de `McpCreateField` ci-dessous / dans
> `MCP-DynamicFunctions-FormBuilder.md` doit inclure cette ligne.

> Les autres fonctions (`McpCreateTable`, `McpCreateField`, `McpCreateEnumField`,
> `McpCreateExpressionField`/`McpSetFieldExpression`, `McpCreateFormulaField`/`McpSetFieldFormula`,
> `McpCreateReverseLink`, `McpBuild`, `McpGrantTablePermission`, `McpSetFieldsApi`,
> `McpSetFieldsUiVisibility`, `McpSetFieldExport`, `McpSetFieldRender`, `McpSetFieldNestedTable`,
> `McpSetFieldUnit`, `McpSetFieldLabel`, `McpSetFieldTexts`, `McpAddFieldsToSection`, `McpSetReference`)
> ont leur **code complet** dans `references/MCP-DynamicFunctions-FormBuilder.md`. Les fonctions
> indicateurs/dashboards/workflow (`McpCreateIndicator`, `McpCreateMultiMeasureIndicator`,
> `McpCreateTypedIndicator`, `McpSetIndicatorChartType`, `McpCreateDashboard`, `McpAddIndicatorWidget`,
> `McpDiagReportData`, `McpCreateEntityWorkflow`, `McpUpdateFunctionCode`) sont dans
> `references/MCP-Indicators-Dashboards.md`.
