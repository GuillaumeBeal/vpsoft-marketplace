# VPSoft — Code dynamique PROPRE : DynamicPages, framework VP (C# & JS), code global, rendu listes

> **But** : écrire du code dynamique VPSoft **aux standards** (framework `VP` de préférence, jamais
> d'accès SQL direct, helpers officiels), de façon **autoportante** (tout le contrat est ici : entités
> verbatim, scopes de variables, pipelines de rendu, fonctions `Mcp*` prêtes). Tout est **vérifié
> sur le source 10.5** (`fichier:ligne`) et **testé sur la démo** (`demo-vesta-10-5`) quand marqué ✅.

**Sommaire** : §0 carte des briques · §1 contrat d'exécution (usings, wrapper, logs) · §2 framework VP
C# · §3 framework VP JS · §4 expressions d'entité (contrat par slot + listeners NHibernate) ·
§5 DynamicPage/Widget/PageField (modèle, rendu, attaches, `McpCreateDynamicPage`/`McpRenderDynamicPage`/
`McpAttachFieldPage` ✅) · §6 code global app & module · §7 slots Content* · §8 rendu listes (jTable,
`RenderExpression*`) · §9 DynamicButton (+ `McpCreateDynamicButton`/`McpSetButtonCode` ✅) ·
§10 Batch/Class/Const (+ `McpCreateBatch`/`McpRunBatch` ✅) · §11 gotchas.

---

## 0. Carte des briques de code (qui vit où, qui s'exécute quand)

| Brique | Entité (table) | Slots de code | S'exécute | Rebuild ? |
|---|---|---|---|---|
| **DynamicFunction** | `DynamicFunction` | `CodeFunction` + `CodeUsing` + `CodeClass` (C#) | Serveur, à l'invocation (compilation à chaud, cache MD5) | Non |
| **Expressions d'entité** | `DynamicSettings` (slots `*Expression`) | 12 slots C# + 15 slots front (Css/Js/Razor) | Serveur (C#) / navigateur (front) | **Oui pour les slots C#** (compilés au build) ; non pour les slots front |
| **DynamicPage / Widget / PageField** | `DynamicPages` (1 table, discriminant `ClassType`) | `CodeRazor` + `CodeCss` + `CodeJavascript` | Serveur (Razor compilé à chaud ou mergé) + navigateur (CSS/JS) | Non (compil. à chaud ; « Merge » optionnel) |
| **Code global (app/module)** | `DynamicGlobalCode` | `CodeRazor` + `CodeCss` + `CodeJavascript` | Toutes les pages du layout dynamique | Non (F5) |
| **DynamicClass** | `DynamicClass` | `CodeClass` + `CodeUsing` (C#) | Référencée par les autres codes dynamiques | Non |
| **DynamicConst** | `DynamicConst` | `Value` (+ `CodeCSharp`) | Cache mémoire, lue par `VP.GetDynamicConst` | Non |
| **DynamicBatch** | `DynamicBatch` | `CodeBatch` + `ClassBatch` + `CodeReferenceBatch` | Serveur, wrappé dans `MainVP()` avec `Context` | Non |
| **DynamicButton** | `DynamicButton` | `DynamicExpression` (C# serveur) + `JSPreCall`/`JSCallBack` (JS client) | Endpoint `RunDynamicButtonOf{Form\|Table}` | Non |

**Règle d'or des standards VPSoft** :
1. **Toujours passer par le framework `VP`** (C# serveur, JS client) plutôt que par NHibernate/SQL/fetch
   bruts — sauf contournement documenté (ex. `McpSetReference`).
2. **Jamais d'exception muette** : `try/catch` qui renvoie/log le message (`VP.SetLogError`,
   `VP.GetLogExceptionInfo`).
3. **Code lisible par le consultant suivant** : noms explicites, une responsabilité par fonction,
   constantes nommées plutôt que littéraux magiques.

---

## 1. Le contrat d'exécution commun (usings, wrapper, logs)

### 1.1 Usings auto-injectés (AUCUN `using` à écrire pour ça)

Tout code dynamique (fonctions, expressions, Razor de pages) est compilé avec la liste
`DynamicCodeBuilderHelper.Usings` (`VPSoft.Services/Helpers/Builder/Codes/DynamicCodeBuilderHelper.cs:94-197`) :
tous les namespaces `VPSoft.*` utiles (`VPSoft.Domain.Models.Builder`, `VPSoft.Domain.Repositories`,
`VPSoft.Domain.Helpers`, `VPSoft.Utils.Extensions` → **la classe `VP`**, `VPSoft.Services.Abstractions`,
`VPSoft.FormBuilder.*` → **les entités dynamiques compilées**…) + `Newtonsoft.Json[.Linq]`,
`Microsoft.AspNetCore.Http`, `System[.Linq|.Collections.Generic|.Threading.Tasks|.IO|.Text|…]`.

⚠️ N'ajouter dans `CodeUsing` QUE les namespaces hors liste (ex. `VPSoft.Domain.Contracts.DynamicPages`,
`VPSoft.Domain.Contracts.Rule.Core`, `VPSoft.Services.Abstractions.Rule`, `VPSoft.Domain.Contracts.VP`).
**Un `using` inexistant ⇒ échec de compilation à chaud ⇒ retour `null` SILENCIEUX.**

### 1.2 Wrapper d'une DynamicFunction (généré, verbatim)

`DynamicCodeBuilderHelper.GenerateStandaloneDynamicFunctionCode` (`:493-553`) :

```csharp
<usings par défaut> + <CodeUsing>
namespace VPSoft.FormBuilder.DynamicFunctions   // BuilderConfiguration.FormBuilderDynamicFunctionsNameSpace
{
    <CodeClass>                                  // classes d'appoint optionnelles
    public static class <Name>…                  // template FormBuilderFunctionsStaticClassNameTemplate
    {
        public static <ReturnValue|void> <Name>(<Parameters>)
        {
            try { <CodeFunction> }
            catch (Exception ex)
            {
                VP.GetLogExceptionInfo("Error => …<Name>", ex, "DynamicFunction");
                return default(<ReturnValue>);   // ← le wrapper AVALE l'exception
            }
        }
    }
}
```

→ D'où la règle de la skill : **try/catch interne** qui retourne `ex.Message` (sinon `null` muet).
Variante async : `GenerateStandaloneDynamicFunctionCodeAsync` (`:589-646`) — retour `Task<System.Object>`.

### 1.3 Logs & diagnostics depuis le code dynamique

| Helper | Usage |
|---|---|
| `VP.SetLogDebug(msg)` / `VP.SetLogError(msg)` (`VP.cs:3275/3284`) | log applicatif (AppLog) |
| `VP.GetLogExceptionInfo(info, ex, entityError)` (`VP.cs:2891`) | log d'exception structuré |
| `VP.GetDynamicFunctionLogCurrent(n)` / `GetDynamicExpressionLogCurrent(n)` / `GetDynamicPageLogCurrent(n)` / `GetDynamicBatchLogCurrent(n)` (`VP.cs:3304-3314`) | lire les N derniers logs par type de code (diagnostic sans accès serveur) |
| MCP : `get_app_errors(search, take)` | même table AppLog côté MCP |

---

## 2. Framework VP — C# serveur (`VPSoft.Utils/Extensions/VP.cs`, `public static class VP` :84)

**LA règle : tout accès données/mail/fichier/UI passe par `VP.*`.** Surface utile (vérifiée) :

### 2.1 `VP.Entities` — CRUD & requêtes (à privilégier)

| Méthode | Signature (essentiel) |
|---|---|
| `Create` / `Edit` | `(IEntity entity, bool refreshExpressionFields = false)` |
| `GetMany` | `(string entityName, string where, string orderBy=null, string fetches=null, string filterQuery="All", params object[] parameters)` + version typée `GetMany<TEntity>(Expression<Func<TEntity,bool>> where, …)` |
| `GetManySelect` | `(string entityName, string select, string where="", …)` + **typée** `GetManySelect<TEntity,TResult>(Expression<Func<TEntity,TResult>> select, Expression<Func<TEntity,bool>> where, …)` |
| `GetSingleSelect` / `GetFirstOrDefault[Select]` | idem en unitaire |
| `GetCount` / `GetSum` / `GetAverage` / `GetMax` / `GetMin` | agrégats (string + typés) |
| `GetEntityFromDate` | `(entityName, entityId, date)` — version historisée (audit) |
| `RefreshExpressionFields(IFormBuilder)` / `RefreshExpressionFieldsAndReturn(entityName, entityId)` | recalcul des champs expression |
| `SetMultiCultureValues(entityName, propertyName, entityId, Dictionary<string,string>)` | valeurs multilingues |

✅ **Exemple réel démo** (widget `TotalInitialBudgetForTheYear`) :
```csharp
decimal total = VP.Entities.GetManySelect<ProgrammationLine, decimal>(
    x => x.Amount,
    x => x.Programmation.Operation.EntityState == EntityState.Active
         && x.Programmation.IsInitial && x.Date.Value.Year == DateTime.Today.Year).Sum();
```

### 2.2 Racine `VP.*` — helpers transverses les plus utiles

| Domaine | Membres (ligne `VP.cs`) |
|---|---|
| Entités (legacy string) | `GetEntity(entityName,id)`:137 · `GetSingle…`:151-195 · `ArchiveEntity`:1038 · `DeleteEntity`:1059 · `SaveEntity`:1090 · `UpdateEntity`:1111 · `GetEntityCount(entityName,filters,filterQuery)`:2953 |
| Utilisateur / contexte | `GetUserCurrent()`:441 (dynamic User) · `VP.Context.HttpContext()/GetCurrentUrl()` · `VP.Date.GetDateTimeNowUser()` |
| Localisation | **`GetResourceDynamic(keyName, cultureCode=null)`:1816** (ressources dynamiques) · `GetCultureParameter`:1786 · `GetTranslatedProperty`:1764 |
| Constantes | `GetDynamicConst(type, key)`:1023 — lit le **cache** alimenté au démarrage (`InitDynamicConstCache`, clé = **`Name` de la const** + "VPDynamicConst") |
| Formatage | `GetFormattedNumber(decimal, symbol=null)`:3620 · `StringToHtml`:3077 · `GetImage(Guid,params)`:3099 / `GetImageBase64`:3114 |
| URLs / liens | `GetEntityUrl(entityName, actionType, id)`:2923 · `GetEntityName(entityName,id)`:2938 · `GetLinkWithURL`:3186 · `GetDownloadFileLink(Guid)` / `GetCachedFileLink(Guid)` |
| **Pages** | **`GetDynamicPage(code, model)`:3203** (rend une DynamicPage, async) · `GetPartialPage(code, model)`:3087 |
| Arbres/périmètres | `GetOrganizationChildren`:1952 · `GetTreeChildren`:2006 · `IsTreeDataChildOf/ParentOf`:2183/2195 · `GetManyPerimeterIn/Of<TEntity>`:371/414 · `VP.Tree.GetPerimeter/GetGlobalFilters/GetTreeWithChildren` |
| AGL (métadonnées) | `VP.AGL.GetLocalizedFieldName/GetLocalizedTableName/GetDynamicFormViewModel/GetDynamicFieldViewModel/GetAllDynamic*FromCache` |
| Fonctions | `VP.Functions.Invoke(functionName, params object[])` / `InvokeAsync` |
| Mail | `VP.Mail.SendMail(subject, body, recipient, templateVarsDico)` · surcharge complète `(recipient, toDisplayName, subject, bodyHTML, bodyPlain, fromDisplayName, attachments?, cc?, bcc?, relatedEntity?, …)` (`using VPSoft.Domain.Contracts.VP` pour `MailFormModel`) |
| Fichiers | `VP.Files.UploadFile(stream\|bytes, fileName, contentType)` · `GetFileBytes/GetFileStream(filePath)` · `GetPDFFromHtml(html, header, footer, landscape)` (async) |
| Temps réel | `VP.Hubs.Notifier.NotifyUser/NotifyAllUsers/NotifyGroup(Notification{Title,Text,Type,Delay})` + `NotifyInElement*` (injection HTML ciblée) |
| Threads | `VP.Thread.ExecuteActionRun(Func<Task>)` / `ExecuteFunctionRun<T>` — exécution parallèle AVEC contexte utilisateur propagé |
| UI | `VP.UI.GetReadableHtmlColor(htmlColor)` · `VP.UI.GetDynamicPage(code, model)` · `VP.UI.GetDynamicListPartial(DynamicListFormModel{EntityName,ViewName})` |

### 2.3 Extensions du **form model** (le standard des expressions `Initialize*`)

Sur `AbstractDynamicFormModel<TEntity>` (donc sur `Model`/`this` dans les expressions — cf. §4) :

| Extension (`VP.cs`) | Effet |
|---|---|
| `Model.OverrideDropDownList(propertyName, …)`:2789/2808 | restreindre/redéfinir la liste déroulante d'un champ référence |
| `Model.AddDefaultValue(propertyName, value)`:2825 · `SetDefaultValue`:2841 | valeur par défaut d'un champ au Create |
| `Model.OverrideValidation(…)`:2860 | surcharger la validation d'un champ |

### 2.4 Helpers de rendu `IVPModel` (utilisables dans les slots **Razor** des champs)

`SetScripts`:2371 · `HideField(vpModel, expression)`:2382 · `SetClasses`:2440/2453 · `SetStyles`:2465/2478 ·
`SetReadOnly`:2489 · `SetDisabled`:2500 · `SetAttributesFromValue/FromId`:2513/2526 ·
`HideIf(operation, property, parameters, vpModel)`:2649 · `HideIfProperty`:2693 ·
`GetEnumInfo/GetEnumResources(entityName, propertyName)`:2734/2762.

---

## 3. Framework VP — JavaScript client (`VPSoft.Web/wwwroot/js/global/api/VP.js`, 538 l.)

Objet global `VP` (ligne 523) : `VP = { Table, Entities, AGL, Context, Functions, User, UI, Url, Files, Mail, Unit, Workflow, Hubs }`.
Toutes les méthodes sont **async** et appellent l'API REST `/Api/V2/VP/...` ; helpers globaux disponibles :
`appBaseURL` (préfixe app), `postAsync/getAsync`, `CheckCustomsErrors`, jQuery `$`.

**Enveloppe de réponse** : `{ Success: bool, Message: string, Result: … }` → toujours tester `res.Success`.

| Classe | Méthodes (signatures JS) |
|---|---|
| `VP.Entities` | `GetMany(entityName, where, orderBy, fetches, filterQuery="All", parameters=[])` · `GetManySelect(entityName, select, where, orderBy, skip, take, filterQuery, parameters)` — **le `select` accepte les alias** `"Id as key, Name as title"` · `GetSingleSelect` · `GetFirstOrDefault[Select]` · `GetCount/GetSum/GetMax/GetMin/GetAverage` · `GetEntityFromDate` · `RefreshExpressionFieldsAndReturn(entityName, entityId)` · `SetMultiCultureValues` · `Create(entityName, entityData, displayLoader=false)` |
| `VP.Functions` | `Invoke(functionName, functionParameters)` → POST `/Api/V2/VP/Functions/Invoke` (payload `{Name, CurrentParameters}`) |
| `VP.AGL` | `GetLocalizedFieldName(entityName, field, viewName)` · `GetLocalizedTableName(entityName)` · `ForceRefreshExpressions(selector, ids)` |
| `VP.UI` | `GetDynamicPage(code, model)` (rend une page côté serveur, retourne le HTML) · `GetDynamicListPartial(entityName, viewName)` · `GetReadableHtmlColor(bg)` |
| `VP.Url` | `GetUrlEntity(entityName, urlAction=0, entityId=null, viewName=null)` (0=Index 1=Create 2=Edit 3=Details) |
| `VP.Files` | `UploadFile(fileFormModel)` · `DownloadFile(filePath)` · `GetPDFFromHtml(html, header, footer, fileName, landScape)` |
| `VP.Mail` | `SendMail(toAddress, toDisplayName, fromDisplayName, subject, bodyHTML, bodyPlain, ccDico, bccDico)` |
| `VP.Workflow` | `SetEntityWorkflowStatus(entityName, entityId, actionId)` — exécuter une transition |
| `VP.Tree` | `GetGlobalFilters(entityName)` · `GetPerimeter(entityName)` · `GetTreeWithChildren(entityName, parentId)` |
| `VP.Unit` | `GetDefaultUnitId/Code/Symbol(unitType)` |
| `VP.User` | `GetAvatarUrl(userId)` |
| `VP.Table` | `$.hik.jtable.prototype` (accès au prototype jTable — cf. §8 rendu listes) |

✅ **Exemple réel démo** (page `PreviewTree`) :
```javascript
VP.Entities.GetManySelect(EntityName, "Id as key,Code,Name as title,Parent.Id as Parent,Level", "EntityState=1")
    .then(res => {
        if (res.Success) {
            let flat = res.Result.sort((a, b) => a.Level - b.Level);
            // … construire l'arbre, init fancytree …
        }
    })
    .catch(err => console.log(err));
```

---

## 4. Expressions d'entité (`DynamicSettings`) — CONTRAT EXACT par slot

Le compilateur `DynamicFormModelCompilator` (`VPSoft.Services/Helpers/Builder/Compilator/DynamicFormModelCompilator.cs`)
génère au **build** une classe `DynamicForm<Entity>Model : AbstractDynamicFormModel<TEntity>` avec
`Model = this` (`:160`) ; chaque slot devient le **corps d'une méthode générée** :

| Slot (`DynamicSettings`) | Méthode générée (verbatim `:188-330`) | Variables en scope | Quand |
|---|---|---|---|
| `InitializeCreateExpression` | `InitializeViewModelCreate()` — **aucun paramètre** | `Model`/`this` (le form model), `VP`, extensions §2.3 | avant rendu du formulaire Create |
| `InitializeEditExpression` / `InitializeDetailsExpression` | `InitializeViewModelEdit()/Details()` | idem (+ entité chargée via le model) | avant rendu Edit/Details |
| `ValidationCreateExpression` | `CustomValidationCreate(ModelStateDictionary modelState)` | `modelState`, `Model` (`Model.Context.GetFormValue("Champ")`), `Submit` | au save, AVANT persist — `modelState.AddModelError("Champ", VP.GetResourceDynamic("msg"))` **rejette** |
| `ValidationEditExpression` | `CustomValidationEdit(modelState)` | idem | idem en édition |
| `InjectionCreateExpression` | `InjectCustomCreate(<EntityType> entity)` | **`entity` (typée)**, `VP` | juste avant persist (Create) |
| `InjectionEditExpression` | `InjectCustomEdit(<EntityType> entity) : bool` | **`entity`**, `VP` | juste avant persist (Edit) |
| `PostCreateExpression` / `PostEditExpression` | `CustomPostCreate/Edit(entity)` | **`entity`** | juste APRÈS persist |
| `ListenerCreateExpression` | — (runtime, voir ci-dessous) | **`entity`** | **NHibernate PostInsert** |
| `ListenerEditExpression` | — (runtime) | **`entity`** | **NHibernate PostUpdate** |
| `UsingsExpression` | usings additionnels partagés par tous les slots | — | — |

> ⚠️ Ces slots C# ont des companions `*Compiled` → **`McpBuild` requis** après modification
> (`McpSetEntityCode` puis build). Les slots front (`Content*Expression`) = F5 seulement.

### 4.1 Les `Listener*Expression` = triggers NHibernate (PAS du « live » formulaire)

`AglExpressionListener : IPostInsertEventListener, IPostUpdateEventListener`
(`VPSoft.Services/Listeners/AglExpressionListener.cs:19-133`) :
- `OnPostInsert` → exécute `ListenerCreateExpression` ; `OnPostUpdate` → `ListenerEditExpression`.
- **Déclenché quel que soit le canal** (UI, API REST/MCP, import si `FullListeners|OnlyDynamicListeners`).
- **Nouveau scope DI + nouvelle session + transaction dédiée** ; l'entité est **rechargée** puis passée
  au code (paramètre **`entity`**, cf. `Consts.Dynamic.Code.DynamicSettingsParameterName = "entity"`,
  `Consts.cs:1159`).
- **Anti-boucle** : la liste des `entity.Id` déjà traités dans la chaîne d'appel court-circuite
  les re-déclenchements (`:66-80`) — un `VP.Entities.Edit` DANS un listener ne re-déclenche pas le
  listener pour la même entité.
- Erreur → log AppLog + **rollback de la transaction du listener seul** (l'écriture d'origine reste).

✅ Exemple réel démo (`MaintenanceTask.ListenerEditExpression`) :
`//MaintenanceTicketHelper.UpdateMaintenanceOrderStatus(entity);`

### 4.2 Recalcul live d'un champ expression (≠ listener)

Le « live » au changement d'un champ du formulaire = `DynamicField.ExpressionTriggerFields`
(`McpSetExpressionTriggers`) — voir SKILL.md §6. Les listeners, eux, tournent **après save**.

### 4.3 Code de transition workflow (rappel)

`EntityWorkflowAction.ValidationExpression` → méthode générée `CustomValidationEdit_Workflow_{actionId}(modelState)` ;
`InjectionExpression` → `InjectCustomEdit_Workflow_{actionId}(entity)` ; **dispatch par le champ `Submit`**
contenant le GUID de l'action (`DynamicFormModelCompilator.cs:203-314`). → `McpSetWorkflowActionCode` + build.

---

## 5. DynamicPage / DynamicWidget / DynamicPageField (pages dynamiques)

### 5.1 Modèle (verbatim `VPSoft.Domain/Models/Builder/DynamicPageBase.cs:19-262`)

- **1 table `DynamicPages`**, discriminant `ClassType` (`DynamicPageMap.cs`) ; 3 sous-classes :
  `DynamicWidget` (enum `DynamicPageType.DynamicWidget=0`), `DynamicPage` (=1), `DynamicPageField` (=2).
- Propriétés clés : `Code` (**unique**, `[EntityCode]`, regex `RegExHelper.CodeDynamicPage`) ·
  `LocalizedName`/`LocalizedDescription` (via `CultureParameters`) · **`CodeRazor` · `CodeCss` ·
  `CodeJavascript`** · `IsIndependent` (true ⇒ rendu dans un **iframe** sandbox) ·
  `RoleInModules : ISet<RoleInModule>` (**qui peut voir la page**) · `CssFiles`/`JavascriptFiles`/
  `DynamicImages : ISet<FileUpload>` (assets) · `DynamicFolder` (rangement) ·
  `IsMergeable`/`MergedDate`/`MergedChecksum`/`CodeCompiled` (merge optionnel dans la DLL).
- **`DynamicPageFile`** (`DynamicPageFile.cs`) : fichiers d'assets autonomes
  (`DynamicPageFileType` : Css=0, Javascript=1, Image=2, Other=3) + `File : FileUpload`.

### 5.2 Rendu (pipeline complet, verbatim)

`DynamicPageBaseService.GetDynamicPage(predicate, modelCurrent, isPreview=false)`
(`VPSoft.Services/Builder/DynamicPageBaseService.cs:720-776`) :
1. Charge le DTO (Code, CodeCss/Razor/Javascript, IsIndependent, RoleInModules).
2. **Contrôle d'accès** : sauf `isPreview`, l'utilisateur doit avoir un rôle ∈ `RoleInModules`
   sinon `UnauthorizedAccessException`. ⚠️ **Une page créée par programme sans `RoleInModules`
   n'est visible par personne en URL directe** (le rendu `isPreview:true` et l'usage en widget interne
   restent possibles).
3. **Razor compilé** : `RazorEngine.InstantiateOrCompileAndRunPage(dynamicPageCode: Code, razorCode, model)`
   (`RazorEngine.cs:271-330`) — si le type existe dans l'assembly mergée → instanciation directe,
   sinon **compilation à chaud**. Classe de base `RazorGeneratorTemplateBase`
   (`RazorEngineOptions.cs:94-243`) : en scope Razor → **`@Model` (dynamic)** + **`@Html` (IHtmlHelper)**
   + tous les usings §1.1 (donc **`VP.*` directement utilisable**).
4. **Composition** (`:765-768`) :
   ```csharp
   string codePage = $@"
       {(!string.IsNullOrEmpty(CodeCss) ? $"<style>{CodeCss}</style>" : "")}
       {codeRazorCompiled}
       {(!string.IsNullOrEmpty(CodeJavascript) ? $"<script>{CodeJavascript}</script>" : "")}";
   ```
5. `IsIndependent=true` ⇒ `<iframe … srcdoc="{HtmlEncode(codePage)}">` (`:773`).

**URLs** (`VPSoft.Web/Areas/Users/Controllers/DynamicPageController.cs`) :
`/{module}/DynamicPage/{code}` · `/DynamicPage/{code}` · `/api/RenderDynamicPage/{code}` —
rendu dans `_Layout-dynamic.cshtml` (donc avec le code global §6).

### 5.3 Points d'attache

| Usage | Mécanisme |
|---|---|
| **Menu** | `NavMenu` de type `MenuType.DynamicPage=1`, sous-classe `NavMenuDynamicPage { DynamicPage }` |
| **Champ de formulaire** | `DynamicField.DynamicPageFieldAdd / Edit / Details` (3 slots → 1 `DynamicPageField` par action) + `RenderType.DynamicPage=16`. Rendu : `DynamicHtmlExtension.GetDynamicPageRenderField` (`VPSoft.Utils/Extensions/DynamicHtmlExtension.cs:582-612`) — **`@Model` = l'entité courante** (Edit/Details) ou les `RequestParameters` (Create). Écriture : `DynamicFieldService` (`:3056-3067`) = `field.DynamicPageFieldX = (DynamicPageField)DynamicPageBaseService.GetSingle(id)` + `RenderTypeOverrided`. → **`McpAttachFieldPage`** ✅ (§5.5) |
| **Widget de dashboard / template visuel** | `DynamicWidget` référencé par `DashboardWidget` / `DynamicColumnTemplate.widgetId` |
| **Depuis du code** | C# : `await VP.GetDynamicPage(code, model)` (`VP.cs:3203`) · JS : `await VP.UI.GetDynamicPage(code, model)` |

### 5.4 Création PAR PROGRAMME — `DynamicPageBaseService.Create(DynamicPageFormModel)` (`:56-119`)

- Vérifie l'**unicité du `Code`** (`Exists(x => x.Code == form.Code)` → Conflict).
- Instancie la bonne sous-classe selon `form.DynamicPageType` (string : `"DynamicPage"` /
  `"DynamicWidget"` / `"DynamicPageField"`).
- Résout `DynamicFolder` : `DynamicModuleId` (→ dossier racine du module) **ou** `DynamicFolderId`.
- `ResourceCultureJsons` requis pour le nom : `ResourceCultureJson { resourceKey = "LocalizedName",
  resourceValues = [ ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = "…" } ] }`
  (`VPSoft.Domain/Contracts/App/ResourceCultureJson.cs`).
- Édition du code après coup : `DynamicPageBaseService.SaveCode(DynamicPageCodeViewModel)`
  (controller AGL : `VPSoft.Presentation/Controllers/DynamicCodes/DynamicPageController.cs`).

### 5.5 ✅ Fonctions MCP validées (code complet — recréables via `McpCreateFunction`)

**`McpCreateDynamicPage`** — params `string name, string code, string pageType, string codeRazor,
string codeCss, string codeJavascript, string isIndependent, string folderId` · CodeUsing :
`using VPSoft.Domain.Contracts.DynamicPages; using VPSoft.Services.Abstractions; using Newtonsoft.Json;`

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var form = new DynamicPageFormModel();
    form.Code = code; form.Name = name; form.DynamicPageType = pageType;
    form.CodeRazor = codeRazor; form.CodeCss = codeCss; form.CodeJavascript = codeJavascript;
    form.IsIndependent = isIndependent == "true";
    form.DynamicFolderId = Guid.Parse(folderId);
    form.ResourceCultureJsons = new List<ResourceCultureJson> {
        new ResourceCultureJson { resourceKey = "LocalizedName", resourceValues = new List<ResourceCultureValueJson> {
            new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = name },
            new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = name } } },
        new ResourceCultureJson { resourceKey = "LocalizedDescription", resourceValues = new List<ResourceCultureValueJson>() } };
    var result = sm.DynamicPageBaseService.Create(form);
    return JsonConvert.SerializeObject(new { success = result.Code == System.Net.HttpStatusCode.OK, id = result.Id, messages = result.Messages });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**`McpRenderDynamicPage`** — param `string code` · CodeUsing : `using VPSoft.Services.Abstractions;`
— rend la page **sans navigateur** (isPreview ⇒ pas de contrôle de rôle) :

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var vm = sm.DynamicPageBaseService.GetDynamicPage(x => x.Code == code, null, true).GetAwaiter().GetResult();
    return vm.CodeCompiled;
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé en conditions réelles (démo)** : création du widget `ZZMcpTestWidget`
(id `4e42cecd-ae96-448d-9565-a140135afcf4`, dossier MCP `d019db8c-…`) avec Razor
`@{ int nb = VP.Entities.GetCount("McpDemoVehicule", "EntityState == EntityState.Active"); }<div…>@nb véhicule(s)…` →
`McpRenderDynamicPage("ZZMcpTestWidget")` retourne exactement
`<style>…</style>\n<div id="zzMcpTestWidget"><h3>3 véhicule(s) actif(s)</h3>…</div>\n<script>…</script>`.
**La boucle créer → compiler → rendre est validée de bout en bout via MCP.**

**`McpAttachFieldPage`** — attacher une page-champ aux slots d'un champ — params
`string entityName, string propertyName, string pageCode, string actions` (`actions` = csv parmi
`Add,Edit,Details`, ou **`clear`** pour tout détacher et restaurer le rendu standard) · CodeUsing : *(aucun)*

```csharp
try {
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    var field = rm.DynamicFieldRepository.GetMany(x => x.EntityName == entityName && x.PropertyName == propertyName).FirstOrDefault();
    if (field == null) return "ERR DynamicField introuvable: " + entityName + "." + propertyName;
    if (actions == "clear") {
        field.DynamicPageFieldAdd = null; field.DynamicPageFieldEdit = null; field.DynamicPageFieldDetails = null;
        field.RenderTypeOverrided = null;
        rm.DynamicFieldRepository.Edit(field);
        return JsonConvert.SerializeObject(new { success = true, cleared = true, field = entityName + "." + propertyName });
    }
    var page = rm.DynamicPageFieldRepository.GetMany(x => x.Code == pageCode).FirstOrDefault();
    if (page == null) return "ERR DynamicPageField introuvable (Code, et type DynamicPageField requis): " + pageCode;
    var acts = (actions ?? "").Split(','); var setList = new List<string>();
    foreach (var a in acts) {
        var act = a.Trim().ToLower();
        if (act == "add") { field.DynamicPageFieldAdd = page; setList.Add("Add"); }
        else if (act == "edit") { field.DynamicPageFieldEdit = page; setList.Add("Edit"); }
        else if (act == "details") { field.DynamicPageFieldDetails = page; setList.Add("Details"); }
    }
    if (setList.Count == 0) return "ERR actions doit contenir Add, Edit et/ou Details (ou clear)";
    field.RenderTypeOverrided = RenderType.DynamicPage;
    rm.DynamicFieldRepository.Edit(field);
    return JsonConvert.SerializeObject(new { success = true, field = entityName + "." + propertyName, pageId = page.Id, pageCode = pageCode, actionsSet = setList, renderType = "DynamicPage(16)" });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé (démo)** : page `ZZMcpTestPageField` (type DynamicPageField, Razor `@Model.Code`) →
`McpAttachFieldPage("McpDemoVehicule","KilometrageAnnuel","ZZMcpTestPageField","Details")` →
relecture `DynamicField` : `RenderTypeOverrided=16` + `DynamicPageFieldDetails.Code="ZZMcpTestPageField"` →
`McpAttachFieldPage(…, "clear")` restaure l'état initial. ⚠️ La page DOIT être de type **DynamicPageField**
(le repo `DynamicPageFieldRepository` ne voit pas les `DynamicPage`/`DynamicWidget`).

### 5.6 Standards d'écriture d'une page/widget propre

1. **Razor = données via `VP.Entities.*` typé** (jamais de SQL, jamais de repository direct).
2. **CSS scopé** : préfixer les sélecteurs par l'id racine du widget (`#monWidget .titre {…}`) pour ne
   pas polluer la page hôte (sinon `IsIndependent=true` + iframe).
3. **JS** : utiliser `VP.*` JS + `appBaseURL` ; pas de variables globales sans préfixe ; `$(document).ready`.
4. **Nom/Code** : `Code` technique stable en PascalCase (il sert d'URL et de clé de compilation), nom
   localisé via cultures.
5. **Toujours poser `RoleInModules`** si la page doit être accessible en URL/menu.

---

## 6. Code global & code global de MODULE (`DynamicGlobalCode`)

### 6.1 Modèle (verbatim `VPSoft.Domain/Models/Builder/DynamicGlobalCode.cs:12-75`)

`DynamicGlobalCode { DynamicModule?, CodeRazor, CodeJavascript, CodeCss, IsJavascriptModule,
IsMergeable, MergedDate, MergedChecksum, CodeCompiled }`
- **`DynamicModule == null` ⇒ code global APPLICATION** (unique).
- **`DynamicModule != null` ⇒ code global du MODULE** (un par module).
✅ Démo : 1 enregistrement app-global + 1 par module (Budget, DataViz, Maintenance, Reporting, CSR…).

### 6.2 Injection (verbatim — où ça atterrit)

`_Layout-dynamic.cshtml` (`VPSoft.Web/Areas/Users/Views/Shared/_Layout-dynamic.cshtml:20,60,80`)
invoque 3× le ViewComponent **`GlobalCodeRendering`** (`VPSoft.Web/Views/Shared/Components/GlobalCodeRendering/`) :

| Part | Où dans la page | Mécanisme |
|---|---|---|
| **CSS** | section `css_override` (head) | `<link rel="stylesheet" href="{app}/Api/UIService/DownloadGlobalFileByType?type=css&date={ModifiedDate}[&moduleCode=…]">` |
| **Razor** | juste **avant `@RenderBody()`** dans `<div id="globalRazor">` | compilé par `RazorEngine.InstantiateOrCompileAndRun(... nameof(DynamicGlobalCode)[_moduleCode], CodeRazor, new object())` (`DynamicModuleService.cs:1977-2015`) — **`@Model` = `new object()` (inutilisable)** ; utiliser `VP.*` |
| **JS** | fin de section `scripts` (après les scripts de la page) | `<script src="{app}/Api/UIService/DownloadGlobalFileByType?type=js&date=…[&moduleCode=…]">` |

- CSS/JS sont servis comme **fichiers virtuels** (`UIServiceController.DownloadGlobalFileByType`,
  cache HTTP **1 an**, invalidé par le paramètre `date=ModifiedDate` → **effet immédiat après save + F5**).
- **Ordre : code app-global PUIS code module** (le module peut surcharger).
- Le module courant est résolu par la **route `{module}`** de l'URL (`GlobalCodeRendering.cs:22`).
- Portée : **toutes les pages utilisant le layout dynamique** (listes, formulaires, pages dynamiques).

### 6.3 Édition par programme

Service `DynamicGlobalCodeService` (`VPSoft.Services/Builder/DynamicGlobalCodeService.cs`) :
`GetCode(Guid? moduleId)`:205 · `TryToSaveCode(DynamicGlobalCodeFormModel, moduleId, …)`:169 ·
`Merge/UnMerge`:259/1048. Création directe possible : `Create(new DynamicGlobalCode { DynamicModule = …, CodeJavascript = … })`.
**Config pure : aucun rebuild, F5 suffit.** (AGL : item `GlobalCode` de l'arbre, onglets JS/CSS/Razor.)

---

## 7. Slots `Content*Expression` (rappel d'injection, par page)

15 slots front sur `DynamicSettings` (`DynamicSettings.cs:243-297`) : `Content{Css|Javascript|Razor}{Create|Edit|Details|List|Global}Expression`.

| Page | Vue qui injecte | Détail |
|---|---|---|
| **Create / Edit / Details** | `_DynamicFormPartial.cshtml` (`VPSoft.Web/Views/Shared/`) | `:26` → `<style>` (Global + action) · `:31` → Razor compilé à chaud avec **`Model.Entity` en scope** · `:1002` → `<script defer>` (Global + action) |
| **Liste (Index)** | `_DynamicListPartial.cshtml:62-68` → `_RenderDynamicCodeList.cshtml` | `DynamicHelper.GetDynamicCodeList(entityName)` (`VPSoft.Utils/Helpers/DynamicHelper.cs:202-221`) **concatène Global + List** (`ContentJavascriptGlobalExpression + " " + ContentJavascriptListExpression`, idem Css/Razor) puis injecte `<style>`, Razor (`InstantiateOrCompileAndRunPage`), `<script>` dans `<div id="DynamicCodeList">` **avant la jTable** |

- Résolution par **`ViewName` d'abord, sinon `EntityName`** (une vue a ses propres slots).
- **`McpSetEntityCode(entityName, slot, code)`** pose n'importe lequel de ces slots (F5, pas de build).

---

## 8. Rendu custom sur les LISTES (jTable)

Les listes d'entités = plugin **jTable** customisé (`VPSoft.Web/wwwroot/js/components/jtable/jquery.jtable.VPSoft.js`).

### 8.1 Points d'extension (verbatim)

1. **`customDisplay` par colonne** (`_createCellForRecordField` `:37-64`) : si la config de colonne
   (`columnExtension` / `JTableColumnJsons`, posée via l'AGL — config jTable de la vue) contient
   `customDisplay`, le code est exécuté par `new Function('data', customDisplay)` avec
   **`data.record` = la ligne** ; retour = contenu/HTML de la cellule :
   ```javascript
   // exemple : "return '<span style=\"color:red\">' + data.record.FirstName + '</span>';"
   ```
2. **Événement `tableReady`** (`recordsLoaded` → `:899-905`) : `CustomEvent` dispatché sur le conteneur
   après chaque chargement —
   ```javascript
   document.querySelector(tableContainer).addEventListener("tableReady", e => { /* e.detail = data */ });
   ```
   C'est **LE hook standard** à utiliser dans `ContentJavascriptListExpression` pour post-traiter le DOM
   de la liste (colorer des cellules, ajouter des badges…), car la jTable recharge en AJAX.
3. **Helpers prototypes** (`SetStyles`:66, `SetStylesRef`:75, `SetContent`:84, `SetScript`:92,
   `SetScriptRecord`:103) : fabriquent des `<td>` stylés/scriptés, utilisables dans `customDisplay`.
4. **`RenderTypeOverrided` / `FieldDisplay`** (config champ, cf. `McpSetFieldRender`) : changement de
   rendu d'un champ SANS code.
5. **Boutons de ligne / d'en-tête** : `DynamicButton` positions `Table=3` / `TableHeader=4` (cf. §9).

### 8.2 Slots de rendu PAR CHAMP — `RenderExpression{Add|Edit|Details|List}` (`DynamicField`)

Chaque champ porte 4 slots de rendu custom (écrits par `DynamicFieldService.cs:3028-3039`) :
- **`RenderExpressionAdd/Edit/Details`** = **Razor** compilé à chaud qui REMPLACE le rendu standard du
  champ au formulaire (`DynamicHtmlExtension.cs:556-562` : `CompileAndRun(razorCode: RenderExpression,
  model: dynamicFieldViewModel)`) — **`@Model` = le `DynamicFieldViewModel`** (valeur, métadonnées,
  helpers `VP.SetStyles/HideIf/SetReadOnly…` §2.4).
- **`RenderExpressionList`** = **JavaScript** côté liste (exporté en `.js` par GitService
  `…/DynamicField/<Prop>_RenderExpressionList.js`).
Posables via PATCH du `DynamicField` (repository) — même famille que `McpSetFieldRender`.

### 8.3 Standard « rendu liste propre »

- Besoin simple (format, unité, libellé) → **config** (`McpSetFieldRender`, unités, labels) — pas de code.
- Mise en forme conditionnelle d'une colonne → `customDisplay` (config jTable de la vue) OU
  `ContentJavascriptListExpression` + `tableReady` (préférer ce dernier via MCP : posable par
  `McpSetEntityCode(entity, "ContentJavascriptListExpression", js)`), OU `RenderExpressionList` par champ.
- Données additionnelles → `VP.Entities.GetManySelect` en JS (jamais de scraping du DOM serveur).

---

## 9. DynamicButton (boutons à code)

### 9.1 Modèle (verbatim `VPSoft.Domain/Models/Builder/DynamicButton.cs:20-119`)

`DynamicButton { LocalizedName, Code, IconName, OrderBy, DynamicExpression /*C# serveur*/,
JSPreCall /*JS avant*/, JSCallBack /*JS après*/, DynamicSettings, DynamicSection?, DynamicField?,
DynamicButtonPosition, FormContext, DisplayLoader, TemplateEntity?, IsSystemButton }`

`DynamicButtonPosition` : `Form=0` · `DynamicSection=1` · `Subsection=2` · `Table=3` (chaque ligne) ·
`TableHeader=4` (toolbar). `FormContext` : sur quels écrans (Create/Edit/Details…).

### 9.2 Exécution (verbatim `VPSoft.Web/Areas/Users/Controllers/DynamicController.cs:1284-1468`)

| Contexte | Endpoint | `Model` en scope dans `DynamicExpression` | Retour |
|---|---|---|---|
| Formulaire | `POST Dynamic/RunDynamicButtonOfForm` | **expando dynamique** = champs du formulaire + valeurs ajoutées par `JSPreCall` (Request.Form) | `{ Id, Result }` |
| Ligne de table | `POST Dynamic/RunDynamicButtonOfTable` (avec `Id`) | **l'entité chargée** (`VP.GetEntity(entityName, id)`) ; si `ExtraData` → expando entité+extras | `{ Id, Ids, Result }` |
| En-tête de table | idem (avec `Ids`) | expando `{ Ids = List<Guid> des lignes sélectionnées }` (+ extras) | `{ Id, Ids, Result }` |

- Le code est exécuté par `CSharpEngineService.GetDynamicMethodResultDynamic` → **paramètre nommé
  `Model`** (`DynamicMethodCompilator.DefaultParameterName = "Model"`, `DynamicMethodCompilator.cs:12`).
- **`return false;` ⇒ HTTP error** (bloque) ; retourner un objet/anonyme ⇒ accessible côté client dans
  `data.Result` du `JSCallBack`.
- `JSPreCall(context)` : JS exécuté AVANT le POST (peut enrichir la requête / annuler).
  `JSCallBack(data)` : JS exécuté avec la réponse.

✅ **Exemple réel démo** (bouton `DeletePFWorkstreamCascade`, position Table) :
```csharp
// DynamicExpression (C#) — Model = l'entité de la ligne
foreach (var item in Model.LookupChildren) {
    VP.DeleteEntity(item.GetType().Name, item.Id.ToString());
}
VP.DeleteEntity(Model.GetType().Name, Model.Id.ToString());
```
```javascript
// JSCallBack — recharge la liste
$(".dynamicListContainer").jtable('load')
```

### 9.3 ✅ Fonctions MCP (créer / coder / supprimer un bouton)

**`McpCreateDynamicButton`** — params `string entityName, string code, string name, string position,
string formContext, string iconName, string orderBy` (`position` = `Form|DynamicSection|Subsection|Table|TableHeader` ;
`formContext` = `None|Detail|Create|Edit` — forcé à None pour Table/TableHeader, logique du service) ·
CodeUsing : *(aucun)* — création par entité directe (mirror `DynamicButtonService.CreateDynamicButton` `:759-804`),
contrôle d'unicité du `Code`, nom localisé fr/en via `CultureParameterService` :

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    if (rm.DynamicButtonRepository.GetCountByFilter(x => x.Code == code) > 0) return "ERR Code de bouton deja existant: " + code;
    var dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    if (dsId == Guid.Empty) return "ERR DynamicSettings introuvable: " + entityName;
    var btn = new DynamicButton();
    btn.DynamicSettings = rm.DynamicSettingsRepository.LoadReference(dsId);
    btn.Code = code;
    btn.OrderBy = string.IsNullOrEmpty(orderBy) ? 99 : int.Parse(orderBy);
    btn.EntityState = EntityState.Active;
    btn.DynamicButtonPosition = (DynamicButtonPosition)Enum.Parse(typeof(DynamicButtonPosition), position);
    btn.FormContext = (btn.DynamicButtonPosition == DynamicButtonPosition.Table || btn.DynamicButtonPosition == DynamicButtonPosition.TableHeader)
        ? VPSoft.Domain.Enums.FormContext.None
        : (VPSoft.Domain.Enums.FormContext)Enum.Parse(typeof(VPSoft.Domain.Enums.FormContext), string.IsNullOrEmpty(formContext) ? "None" : formContext);
    if (!string.IsNullOrEmpty(iconName)) btn.IconName = iconName;
    bool ok = sm.DynamicButtonService.Create(btn);
    var rc = new ResourceCultureJson { resourceKey = "LocalizedName", resourceValues = new List<ResourceCultureValueJson> {
        new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = name },
        new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = name } } };
    sm.CultureParameterService.SaveOrUpdate(rc, btn.Id, typeof(DynamicButton).GetProperty("LocalizedName"));
    return JsonConvert.SerializeObject(new { success = ok, buttonId = btn.Id, code = code, position = position, formContext = btn.FormContext.ToString() });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

> ⚠️ **`FormContext` DOIT être pleinement qualifié** (`VPSoft.Domain.Enums.FormContext`) : le nom simple
> est **ambigu à la compilation à chaud** (collision avec un type homonyme d'une assembly chargée) ⇒
> retour `null` silencieux. Cf. gotcha §11.

**`McpSetButtonCode`** — params `string buttonCode, string dynamicExpression, string jsPreCall, string jsCallBack`
(chaîne vide ⇒ slot remis à null) · CodeUsing : *(aucun)* :

```csharp
try {
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    var btn = rm.DynamicButtonRepository.GetMany(x => x.Code == buttonCode).FirstOrDefault();
    if (btn == null) return "ERR DynamicButton introuvable (Code): " + buttonCode;
    btn.DynamicExpression = string.IsNullOrEmpty(dynamicExpression) ? null : dynamicExpression;
    btn.JSPreCall = string.IsNullOrEmpty(jsPreCall) ? null : jsPreCall;
    btn.JSCallBack = string.IsNullOrEmpty(jsCallBack) ? null : jsCallBack;
    rm.DynamicButtonRepository.Edit(btn);
    return JsonConvert.SerializeObject(new { success = true, buttonId = btn.Id, code = buttonCode,
        hasServerCode = btn.DynamicExpression != null, hasPreCall = btn.JSPreCall != null, hasCallBack = btn.JSCallBack != null });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**`McpDeleteDynamicButton`** — param `string buttonCode` · `sm.DynamicButtonService.Delete(btn.Id)` (même
résolution par Code que ci-dessus).

✅ **Testé (démo)** : `McpCreateDynamicButton("McpDemoVehicule","ZZTestBtnMcp","ZZ Test bouton MCP",
"TableHeader","None","mdi-rocket-launch","99")` → `McpSetButtonCode("ZZTestBtnMcp",
"VP.SetLogDebug(\"…ids=\" + Model.Ids.Count); return new { Msg = \"hello…\" };", "",
"console.log('ZZTestBtnMcp', data.Result.Msg); $(\".dynamicListContainer\").jtable('load');")` →
relecture : `DynamicButtonPosition=4`, LocalizedName localisé, slots posés → `McpDeleteDynamicButton` ✅.

---

## 10. DynamicBatch / DynamicClass / DynamicConst

### 10.1 DynamicBatch (`BatchBase.cs:11-44`)

`DynamicBatch : BatchBase { Name, Code, IsAvailableUserSide, Description, CodeBatch, ClassBatch, CodeReferenceBatch, DynamicFolder }`.
**Contrat d'exécution** (`CSharpEngineService.CompileCSharpBatch` `:541-649`, verbatim) :
```csharp
namespace dynamiccompilation {
    <ClassBatch>
    public class <prefix> {
        private CSharpEngineService.BatchContext Context;          // ← INJECTÉ (constructeur)
        public void MainVP() { <CodeBatch> }                        // ← votre code
    }
}
```
En scope : **`Context`** (`IBatchContext`, `VPSoft.Domain/Interfaces/IBatchContext.cs` :
`AddLog(string)`, `SetReturnMessage(string)`, `SetProgress(message, percentage)`, `GetLogs()`,
`GetReturnMessage()`, `CancelProcess()`, `GetPercentProgress()` — + `CancellationToken`/`Token`/`BatchId`/`Data`
sur le `BatchContext` concret) + usings §1.1 (donc `VP.*`).
Standard : tester `Context.CancellationToken.IsCancellationRequested` dans les boucles longues,
journaliser par `Context.AddLog`, conclure par `Context.SetReturnMessage`.

**Exécution** : `DynamicBatchController.ExecuteBatch(id)` (`VPSoft.Presentation/Controllers/DynamicCodes/`)
construit `BatchProcessData(token, name, batchId, user)` (`VPSoft.Domain/Utils/Process/Utils/ProcessData.cs:508`)
→ `CSharpEngineService.CompileCSharpBatch(CodeBatch, processData, ClassBatch, CodeReferenceBatch)` →
journalise un **`ExecutedTask`** (historique du batch : IsSuccess, Message=ReturnMessage, Logs, dates, Planner).
Planification : `DynamicBatchService.CreateScheduledTask` → `BatchScheduledTask { TaskType=DynamicBatch }`
(écran ScheduledTasks). Diagnostic : `VP.GetDynamicBatchLogCurrent(n)`.

#### ✅ Fonctions MCP (créer / exécuter un batch)

**`McpCreateBatch`** — params `string name, string description, string codeBatch, string classBatch,
string codeReferenceBatch, string folderId, string isAvailableUserSide` · CodeUsing : *(aucun)* —
unicité du **Name** (règle du controller), `Code` = GUID auto :

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    if (sm.DynamicBatchService.GetCountByFilter(x => x.Name == name) > 0) return "ERR Nom de batch deja existant: " + name;
    var batch = new DynamicBatch();
    batch.Name = name; batch.Code = Guid.NewGuid().ToString(); batch.Description = description;
    batch.CodeBatch = codeBatch;
    batch.ClassBatch = string.IsNullOrEmpty(classBatch) ? null : classBatch;
    batch.CodeReferenceBatch = string.IsNullOrEmpty(codeReferenceBatch) ? null : codeReferenceBatch;
    batch.IsAvailableUserSide = isAvailableUserSide == "true";
    batch.EntityState = EntityState.Active;
    batch.DynamicFolder = rm.DynamicFolderRepository.LoadReference(Guid.Parse(folderId));
    bool ok = sm.DynamicBatchService.Create(batch);
    return JsonConvert.SerializeObject(new { success = ok, batchId = batch.Id, name = name, code = batch.Code });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**`McpRunBatch`** — param `string nameOrCode` · **CodeUsing : `using VPSoft.Domain.Utils.Process.Utils;`**
(`BatchProcessData` n'est PAS dans les usings par défaut) — exécution synchrone, renvoie message + logs :

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var batch = sm.DynamicBatchService.GetMany(x => x.Name == nameOrCode || x.Code == nameOrCode).FirstOrDefault();
    if (batch == null) return "ERR DynamicBatch introuvable (Name ou Code): " + nameOrCode;
    if (string.IsNullOrWhiteSpace(batch.CodeBatch)) return "ERR CodeBatch vide pour: " + nameOrCode;
    var processData = new BatchProcessData(Guid.NewGuid().ToString(), batch.Name ?? batch.Code, batch.Id, sm.UserService.GetCurrent());
    var res = sm.CSharpEngineService.CompileCSharpBatch(batch.CodeBatch, processData, batch.ClassBatch ?? "", batch.CodeReferenceBatch ?? "").GetAwaiter().GetResult();
    if (res.HasError) { var errs = new List<string>(); foreach (var l in res.Logs) errs.Add(l.Message);
        return JsonConvert.SerializeObject(new { success = false, compilationErrors = errs }); }
    var logs = processData.Context != null ? processData.Context.GetLogs() : new List<string>();
    var msg = processData.Context != null ? processData.Context.GetReturnMessage() : null;
    return JsonConvert.SerializeObject(new { success = true, batch = batch.Name, returnMessage = msg, logs = logs });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé (démo)** : `McpCreateBatch("ZZ_McpTestBatch", …, "int nb = VP.Entities.GetCount(\"McpDemoVehicule\",
\"EntityState == EntityState.Active\"); Context.AddLog(\"Vehicules actifs: \" + nb); Context.SetProgress(\"comptage fait\",
100); Context.SetReturnMessage(\"OK - \" + nb + \" vehicule(s) actif(s)\");", "", "", "<folderId>", "false")` →
`McpRunBatch("ZZ_McpTestBatch")` → `{ success:true, returnMessage:"OK - 3 vehicule(s) actif(s)",
logs:["Vehicules actifs: 3"] }`. **La boucle créer → compiler (wrapper MainVP/Context) → exécuter est validée.**
⚠️ `McpRunBatch` n'écrit PAS la ligne d'historique `ExecutedTask` (réservée au chemin UI/scheduler).

### 10.2 DynamicClass (`DynamicClass.cs:15-119`)

`{ Name, Code, CodeClass, CodeUsing, CodeCompiled, IsMergeable, … }` — compilée dans
`namespace VPSoft.FormBuilder.DynamicClasses` (`GenerateStandaloneDynamicClassCode` `:515-527`).
Les autres codes dynamiques l'utilisent **directement par nom de classe** (namespace auto-importé §1.1).
Standard : y factoriser la logique partagée entre fonctions/expressions (ex. `MaintenanceTicketHelper`).

### 10.3 DynamicConst (`DynamicConst.cs:10-50`)

`{ Name, Code, CodeCSharp, Description, ConstType (Text=0, Session=1, Cache=2), Value, DynamicFolder }`.
Chargées **au démarrage** dans le cache (`InitDynamicConstCache`, `CSharpEngineService.cs:700-715`,
clé = **`Name`** + `"VPDynamicConst"`). Lecture : **`VP.GetDynamicConst(type, key)`** (`VP.cs:1023`,
`key` = le **Name** ; le 1er paramètre est ignoré). ⚠️ Une const créée/modifiée n'est visible qu'après
rechargement du cache (reboot/refresh cache).

---

## 11. Gotchas spécifiques code (complément SKILL.md §6)

- **`GetDynamicPage` hors preview exige un rôle** : poser `RoleInModules` sur la page sinon
  `UnauthorizedAccessException` en URL directe (cf. mémoire incidents : symptôme « unauthorized »).
- **Razor global (code global) n'a PAS de model** (`new object()`) — tirer les données via `VP.*`.
- **`Listener*` ≠ live formulaire** : post-insert/update NHibernate (tout canal), anti-boucle par Id,
  session dédiée. Le live formulaire = `ExpressionTriggerFields`.
- **`Model` (boutons, dynamic) vs `entity` (expressions, typée)** : ne pas se tromper de nom de variable.
- **Slots C# `DynamicSettings`/workflow = compilés au BUILD** (`*Compiled`) → `McpBuild` après pose ;
  slots front + pages + global code + fonctions = à chaud (F5).
- **`async` dans une DynamicFunction sync** : terminer par `.GetAwaiter().GetResult()`
  (cf. `McpRenderDynamicPage`) — jamais `.Result` sur un contexte qui peut deadlocker… (pattern validé).
- **⚠️ Noms de types AMBIGUS à la compilation à chaud ⇒ `null` silencieux (vécu).** Le compilateur
  charge ~80 usings + toutes les assemblies du bin : certains noms simples deviennent ambigus.
  **Cas avéré : `FormContext`** (`var x = FormContext.None;` seul ⇒ échec de compilation muet) →
  **qualifier complètement** : `VPSoft.Domain.Enums.FormContext.None`. En cas de retour `null`
  inexpliqué : **bissecter** avec `McpUpdateFunctionCode` (corps `return "pong";` puis réintroduire
  les lignes une à une — méthode validée pour isoler la ligne fautive en 3-4 itérations).
- **Erreur 500 + page HTML « Transaction not connected, or was disconnected »** : hoquet transactionnel
  du serveur de démo (filtre NHibernate) — l'écriture est rollbackée. **Vérifier l'absence d'objet
  partiel puis simplement réessayer** (validé : la même requête passe au 2ᵉ essai).
- **CSS/JS global cachés 1 an** : l'URL change avec `ModifiedDate` → toujours sauver via le service
  (sinon cache navigateur obsolète).
- **`IsIndependent`** : iframe = isolation CSS/JS totale, mais pas d'accès au DOM parent ni à `VP` du parent.

---

## 12. Pages INTERACTIVES (cockpit CRUD) — patterns validés ✅ (session 2026-06)

But : une DynamicPage « cockpit » où l'utilisateur crée/lie/consulte tout sans changer d'écran (réf. : page
`BdgCockpit` « Pilotage budgétaire », module Bdg — KPIs live + cartes + création enveloppe/engagement/dépense en 1 clic).

### 12.1 Surface client `VP` JS (`wwwroot/js/global/api/VP.js` ; retour `{Success, Message, Result}`)
- **Lecture (OK 100% client)** : `VP.Entities.GetManySelect(entityName, select, where, orderBy, skip, take)`,
  `GetSum(entity, select, where)`, `GetCount`, `GetSingleSelect`, `GetFirstOrDefaultSelect`. Select imbriqué
  `"Envelope.Code"` → clé **aplatie `EnvelopeCode`** ; `where` multi-niveau OK (`Envelope.FiscalYear.Code == "X"`).
- `VP.Functions.Invoke(name, paramsArray)` → POST `/Api/V2/VP/Functions/Invoke` (`{Name, CurrentParameters}`),
  `r.Result` = la valeur de retour de la DynamicFunction.
- ⚠️ **`VP.Entities.Create(entityName, dataObj)` NE LIE PAS LES RÉFÉRENCES.** `VP2Controller.Create:312-346` fait
  `prop.SetValue(e, Convert.ChangeType(value, prop.PropertyType))` → **échoue silencieusement** (try/catch loggé)
  sur tout champ référence (code string → type entité impossible) et souvent decimal/enum reçus en JsonElement →
  **seuls les `string` sont écrits**. Donc inutilisable pour créer une entité liée (enveloppe→exercice, engagement→enveloppe…).

### 12.2 CRÉER depuis une page → fonction serveur via `VP.Functions.Invoke`
Helper serveur (réflexion + NHibernate session) qui résout les réfs par `Code`, parse enum/decimal/date, pose
`Code`+audit → **`BdgCreate(entityName, fieldsJson)`** (code complet dans `BOOTSTRAP.md`). Côté page JS :
```javascript
var r = await VP.Functions.Invoke("BdgCreate", [entity, JSON.stringify(data)]);
var res = (r && r.Result !== undefined) ? r.Result : r;
if (res && res.success) { /* toast + refresh */ }
```
`data` : réf = **code direct** (`{FiscalYear:"FY2026", Envelope:"ENV-…"}`), enum = **int** (`{BudgetType:1, Status:2}`).
(Alternative : l'outil MCP `create_entity` = API **v1** — gère les réfs (champ direct=code, PAS `_Code_`) **si**
`DynamicFieldRole.Api` exposé via `McpSetFieldsApi` ; effet après rafraîchissement du cache `[EntityCache]`. Chemin
DIFFÉRENT de l'endpoint V2 navigateur ci-dessus.)

### 12.3 ⚠️ Page PLEINE (via menu) : `VP` pas encore défini au chargement → `VP is not defined`
Layout `DynamicPage/Index.cshtml` → `_Layout-dynamic` → `_Layout-user` : le bundle JS (dont `VP.js`, cf.
`bundleconfig.json`) est rendu **EN BAS** (section `scripts`), APRÈS le `<script>` inline de la page (injecté via
`@RenderBody`). L'IIFE de la page tourne donc avant que `const VP` existe. → **différer l'init par un poll** :
```javascript
var n=0;(function boot(){ if(typeof VP==="undefined"||!VP.Entities){ if(++n>200)return; setTimeout(boot,50); return; } init(); })();
```
(Les **widgets de carte de liste** n'ont pas ce souci : chargés en AJAX par `/Users/Page?id=` APRÈS `VP`.)

### 12.4 Transférer un gros bloc (Razor/CSS/JS) sans casser l'échappement
L'échappement JSON manuel d'~10 KB est très fragile (1 caractère corrompu = tout cassé ; vécu). FIABLE : **base64** via
**`McpSetDynamicPageJsB64`** (code dans `BOOTSTRAP.md`). Procédure : `base64 -i f.js | tr -d '\n'`, découper en ~3 parts,
**`cat` UN chunk à la fois** (la sortie shell = exactement ce chunk → impossible de déborder sur le suivant), envoyer
1ᵉʳ `reset="true"` … dernier `finalize="true"` (le helper accumule puis décode). Toujours `node --check` le JS AVANT,
et `McpRenderDynamicPage` APRÈS (le rendu inclut le JS verbatim sans l'exécuter → confirme le round-trip).
⚠️ Le `total` renvoyé = `string.Length` C# (chars) < taille octets si accents/€/⚠ : écart NORMAL, pas une perte.

### 12.5 Mettre la page au menu + rôles d'accès
1. `McpSetDynamicPageRoles(code, "VPWAdmin,Reader")` — sinon `UnauthorizedAccessException` (DynamicPageBaseService:745).
2. `McpAddPageToMenu(pageCode, moduleId, name, label, icon, order, roleCodesCsv)` → `NavMenuDynamicPage`
   (`MenuType.DynamicPage=1`) via `NavMenuService.ActivateOrCreate`. Réordonner les autres entrées avec
   `McpSetNavMenuIconOrderBulk`. (Codes complets : `BOOTSTRAP.md`.)
