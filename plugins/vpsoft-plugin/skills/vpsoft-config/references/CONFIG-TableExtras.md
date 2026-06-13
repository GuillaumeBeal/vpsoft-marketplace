# VPSoft — Onglets avancés d'une table : VISUELS (cards), CONFIDENTIALITÉ, PDF, ALERTES, AIDES

> Les onglets de l'écran AGL d'une table non couverts ailleurs. Tout est **vérifié sur le source 10.5**
> (`fichier:ligne`) et **testé sur la démo** quand marqué ✅. Fonctions `Mcp*` prêtes (code complet).
> (Workflow → `MCP-Indicators-Dashboards.md` ; Boutons → `CODE-DynamicPages-VPFramework.md` §9.)

**Sommaire** : §1 Visuels (widgets avant/après table + **carte cliquable qui préfiltre**) ·
§2 Confidentialité · §3 PDF · §4 Alertes (DataAlert) · §5 Aides (HelpOnline) · §6 gotchas.

---

## 1. VISUELS — widgets/cards au-dessus et en-dessous des listes

### 1.1 Modèle (verbatim)

**`DynamicSectionTemplate`** (`VPSoft.Domain/Models/Builder/DynamicSectionTemplate.cs:1-78`) — une
RANGÉE de cards, **par rôle** :
`{ Code, DynamicSettings, Role (RoleInModule), IsBeforeTable (true=AVANT la liste), Position,
DynamicColumnTemplateList : ISet<DynamicColumnTemplate> }`

**`DynamicColumnTemplate`** (`DynamicColumnTemplate.cs:1-101`) — une CARD :
`{ Code, ContentType, Content, Position, ClassWidth ("col", "col-md-4", "col-lg-3"…), File?,
LocalizedContents (CultureParameter pour le texte multilingue) }`
**`DynamicColumnContentType`** : `Empty=0, Text=1` (HTML), **`Widget=2`** (`Content` = **Id du
DynamicWidget en string**), `Image=3`, `File=4` (via `FileUpload`).

### 1.2 Rendu (verbatim — qui voit quoi, où)

Page liste `VPSoft.Web/Areas/Users/Views/Dynamic/Index.cshtml:58-141` : AJAX
`GET Dynamic/GetCToolsTemplateConfigByRole?entityName=<EntityName|ViewName>`
(`DynamicController.cs:569-597`) — **filtré sur LES RÔLES DE L'UTILISATEUR COURANT**
(`roleInModuleIds.Contains(x.Role.Id)`). Chaque section devient `<div class="row">` injectée dans
`#dynamicListDashboardTop` (IsBeforeTable) ou `#dynamicListDashboardBottom` ; chaque colonne :
- `Text=1` → `innerHTML = Content` (le contenu **localisé**) ;
- **`Widget=2` → `$(col).load(appBaseURL + '/Users/Page?id=' + Content)`** (rendu serveur du
  DynamicWidget : Razor + CSS + JS, cf. CODE-DynamicPages §5) ;
- `Image=3` → `GetImageBase64?id=FileId` ; `File=4` → lien `DownloadFile?id=FileId`.

### 1.3 ✅ Fonctions MCP

**`McpAddTableWidget`** — params `string entityName, string roleCode, string isBeforeTable,
string position, string columnsJson` · CodeUsing : *(aucun)* ·
`columnsJson` = `[{"type":"text"|"widget","content":"<h3>…</h3>"|"<CodeOuIdDuWidget>","classWidth":"col-md-4"}, …]`

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    var ds = rm.DynamicSettingsRepository.GetMany(x => x.EntityName == entityName && x.ViewName == null).FirstOrDefault();
    if (ds == null) return "ERR DynamicSettings introuvable: " + entityName;
    // ⚠️ RoleInModule est PAR MODULE : résoudre le rôle DANS le module de l'entité (sinon on attrape
    // le 1ᵉʳ rôle homonyme d'un AUTRE module → invisible dans l'admin Visuels, cf. §1.3 gotcha #2).
    var moduleCode = sm.DynamicModuleService.GetDynamicModuleCodeFromEntityName(entityName);
    var role = rm.RoleInModuleRepository.GetMany(x => x.Code == roleCode && x.DynamicModule.Code == moduleCode).FirstOrDefault();
    if (role == null) return "ERR RoleInModule introuvable (Code '" + roleCode + "' dans le module '" + moduleCode + "' de l'entite " + entityName + ")";
    var section = new DynamicSectionTemplate();
    section.Code = "MCP_" + Guid.NewGuid().ToString("N").Substring(0, 8);
    section.DynamicSettings = ds; section.Role = role;
    section.IsBeforeTable = isBeforeTable != "false";
    section.Position = string.IsNullOrEmpty(position) ? 0 : int.Parse(position);
    section.DynamicColumnTemplateList = new HashSet<DynamicColumnTemplate>();
    var arr = JArray.Parse(columnsJson); int pos = 0; var colInfos = new List<string>();
    foreach (var it in arr) {
        var col = new DynamicColumnTemplate();
        col.Code = "MCPCOL_" + Guid.NewGuid().ToString("N").Substring(0, 8);
        col.Position = pos; col.ClassWidth = (string)(it["classWidth"] ?? "col");
        var type = ((string)(it["type"] ?? "text")).ToLower();
        if (type == "widget") {
            string wRef = (string)it["content"]; Guid wId;
            if (!Guid.TryParse(wRef, out wId)) {
                var w = rm.DynamicPageBaseRepository.GetMany(x => x.Code == wRef).FirstOrDefault();
                if (w == null) return "ERR widget introuvable (Code ou Id): " + wRef;
                wId = w.Id;
            }
            col.ContentType = DynamicColumnContentType.Widget; col.Content = wId.ToString();
            colInfos.Add("widget:" + wId);
        } else {
            col.ContentType = DynamicColumnContentType.Text; col.Content = (string)(it["content"] ?? "");
            colInfos.Add("text");
        }
        col.DynamicSectionTemplate = section; section.DynamicColumnTemplateList.Add(col); pos++;
    }
    bool ok = sm.DynamicSectionTemplateService.Create(section);
    return JsonConvert.SerializeObject(new { success = ok, sectionId = section.Id, sectionCode = section.Code,
        entityName = entityName, roleCode = roleCode, isBeforeTable = section.IsBeforeTable, columns = colInfos });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**`McpDeleteTableWidget`** — param `string sectionCode` →
`sm.DynamicSectionTemplateService.Delete(section.Id)` (résolution par Code ; cascade colonnes).

✅ **Testé (démo)** : `McpAddTableWidget("McpDemoVehicule","<RoleInModule.Code VPW Admin>","true","1",
'[{"type":"text","content":"<div class=\"text-center\"><h4>Parc véhicules</h4>…</div>","classWidth":"col-md-4"},
{"type":"widget","content":"ZZMcpTestWidget","classWidth":"col-md-4"}]')` → section `MCP_13f20344`
vérifiée en base (Text + Widget) — **visible au-dessus de la liste McpDemoVehicule** pour le rôle.
⚠️ Le `roleCode` = `RoleInModule.Code` (souvent un GUID-like, ex. `228B4CB4-…`) ; un même Code peut
exister dans plusieurs modules. La carte n'apparaît que pour les **utilisateurs ayant ce rôle**.

#### ⚠️ Gotchas VISUELS (incidents réels — démo Bdg)

1. **`classWidth` & retour à la ligne** : le rendu (`Dynamic/Index.cshtml:95`) crée la section en
   **`<div class="row mx-0 flex-wrap gap-2">`** et chaque colonne en
   **`<div class="${ClassWidth} bg-white rounded p-3 …">`** (ligne 100). Le `gap-2` (0,5rem entre
   colonnes) **s'ajoute** à la largeur des colonnes. Donc **N cartes en `col-md-(12/N)` débordent** :
   4×`col-md-3` = 100% **+ 3 gaps de 0,5rem ⇒ la 4ᵉ carte passe à la ligne (3+1)**. ✅ **Solution :
   utiliser `"col"`** (flex-fill : les N cartes se partagent `largeur − gaps`, toutes sur **une seule
   ligne**). C'est **déjà le défaut** de `McpAddTableWidget` (`it["classWidth"] ?? "col"`) → **ne PAS
   forcer `col-md-N`** pour une rangée de quickfilters ; omettre `classWidth` (ou mettre `"col"`).
   *Retrofit d'une section existante* → **`McpSetColumnWidth(sectionId, "col")`** (cf. BOOTSTRAP.md) :
   passe toutes les colonnes d'une section à un `ClassWidth` donné (NHibernate direct ; aucun cache sur
   `DynamicSectionTemplate`/`DynamicColumnTemplate` → effet immédiat au prochain chargement, F5).

2. **Onglet admin VISUELS = filtré PAR RÔLE SÉLECTIONNÉ** (≠ liste utilisateur). La liste **utilisateur**
   (`Dynamic/Index.cshtml:78` → `AGL/GetCToolsTemplateConfigByRole`) affiche les sections des **rôles de
   l'utilisateur courant**. Mais l'onglet **admin** « Visuels » appelle
   `CToolsController.GetCToolsTemplateConfigByRole(dynamicSettingsId, **roleId**)` →
   `DynamicSectionTemplateService.GetDynamicSectionTemplateFormModelList(ds, roleId)` qui filtre
   `x.Role.Id == roleId` (`DynamicSectionTemplateService.cs:238`). Le **roleId** vient du sélecteur de
   rôle de l'onglet, peuplé par `RoleService.GetAllRolesInModuleFromEntity(entityName)`
   (`CToolsController.cs:291`) ⇒ **les `RoleInModule` du module de l'entité** uniquement.

   ⚠️ **PIÈGE RACINE (`RoleInModule` est PAR MODULE)** : un même `Code` (ex. `VPWAdmin`) existe **une fois
   par module** (Bdg, Legal, Portfolio…), chacun avec un `Id` différent. L'ancienne `McpAddTableWidget`
   résolvait le rôle par `GetMany(x => x.Code == roleCode).FirstOrDefault()` (**global**) → elle attrapait
   le 1ᵉʳ homonyme (souvent un AUTRE module). Résultat **observé** : la section **s'affiche bien sur la
   liste utilisateur** (le rendu filtre sur `roleInModuleIds.Contains(x.Role.Id)` = TOUS les rôles de
   l'user, donc le mauvais module passe quand même) **mais reste INTROUVABLE dans l'admin Visuels même en
   sélectionnant le bon nom de rôle** (l'admin envoie le `RoleInModule.Id` du **bon module**, qui ≠ celui
   stocké). ✅ **Fix appliqué** : `McpAddTableWidget` résout désormais le rôle **dans le module de
   l'entité** (`x.Code == roleCode && x.DynamicModule.Code == moduleCode`, via
   `DynamicModuleService.GetDynamicModuleCodeFromEntityName`). ✅ **Retrofit d'une section existante mal
   rattachée** : `McpSetSectionRole(sectionId, <RoleInModule.Id du BON module>)` (cf. BOOTSTRAP.md).
   Diagnostic : comparer `get_many_select("DynamicSectionTemplate","Role.Id,Role.Code","Id==Guid(\"…\")")`
   au `get_many_select("RoleInModule","Id,DynamicModule.Code","Code==\"VPWAdmin\"")` (repérer la ligne du
   module de l'entité).

3. **Nommage & rangement des widgets de carte.** Dans la grille de config Visuels, la **carte d'un widget
   affiche l'`Id` (GUID) du DynamicWidget** (`ctools.js` pose `column.Content`) — c'est le rendu de la SPA
   admin compilée, **non modifiable par config**. Le **nom** du widget (`LocalizedName`) apparaît en
   revanche dans le **sélecteur** de la modale d'édition de la carte (`DynamicPageList`) et dans
   **l'arborescence des pages/widgets**. ⇒ donner des `LocalizedName` clairs aux widgets (sinon le
   consultant ne s'y retrouve pas). ⚠️ **Les widgets créés via MCP sont rangés dans le dossier `API MCP`
   du module `System`** → les **déplacer dans un dossier du module métier** : `McpCreateFolderMovePages`
   (cf. BOOTSTRAP.md) crée un dossier nommé sous la racine du module (`McpListModuleFolders` pour trouver
   la racine, `parent:"ROOT"`) et y déplace les widgets par Code. ✅ Vérif : `get_many_select(
   "DynamicPageBase","Code,LocalizedName,DynamicFolder.LocalizedName,DynamicFolder.DynamicModule.Code",
   "Code.StartsWith(\"…\")")`. Déplacer un widget ne change ni son `Id` ni ses `RoleInModules` → le rendu
   des cartes et le préfiltre restent intacts.

### 1.4 Pattern consultant : CARTE KPI CLIQUABLE QUI PRÉFILTRE la liste ⭐

Combo : un **DynamicWidget** (KPI Razor via `VP.Entities.GetCount`) + un **JS au clic** qui applique un filtre.

🔴 **LE BON PATTERN = persister le filtre côté SERVEUR (session) puis recharger.** ⚠️ **NE PAS** se contenter
d'un `$(".dynamicListContainer").jtable("load",{jsonFilters})` côté client : ça filtre l'affichage mais **ne
persiste PAS au F5** et **n'affiche NI le bandeau de filtres NI le compteur**. Mécanisme natif (repris de
`dynamicFilters.js:saveAndApplyFilter`), **validé end-to-end prod 10.5** :

1. **Persister** : `POST /Api/DynamicFilters/SetFiltersSession {EntityName, ViewName, JsonFilters}` →
   `FilterHelper.SetFilters` stocke en **session** (`DynamicFiltersController.cs:41`). « Tout effacer » =
   `POST /Api/DynamicFilters/ResetTableFilterSession {EntityName, ViewName}`.
2. **Rafraîchir SANS recharger la page** (préféré — validé) : `$('.dynamicListContainer').jtable('load')` (sans
   argument → relit la session côté serveur, `reLoadAfterFilterChange`) PUIS reconstruire le **bandeau de chips +
   le compteur** comme `saveAndApplyFilter` (`:272-317`). 🔴 **GOTCHA** : pour rendre le bandeau visible, faire
   **`bannerEl.classList.remove('d-none')`** sur `[entity-filters-banner]` — **NE PAS ajouter `d-flex`** (ça
   transforme l'élément en flex → le conteneur interne tombe à ~380px → `updateFiltersBanner` croit que le chip
   déborde et l'affiche en « +1 » au lieu de l'afficher inline). Reset = `bannerEl.classList.add('d-none')`.
   Code complet ci-dessous. *(Alternative simple mais avec flash : `location.reload()` — au chargement le serveur
   rerend liste+bandeau+compteur depuis la session, `global/filters.js:673-707` ; c'est ce que fait le filtre
   sauvegardé natif `applySaveFilter`.)*

⚠️ **FORMAT DU FILTRE = sortie de `serializeFilters`** (ce que le modal natif POST ; à matcher pour que le chip
s'affiche). Lire la vraie ligne de filtre dans le DOM : `[filters-table] div[id*="opts_"]` donne
`entityPropertyName, displayName, valueType, isKeysValues, isForeignKey, propertyType` ; la `<select values_>`
donne les valeurs (= **TechnicalName**, ex. `Ordered`). Pour un **enum multi-select** (`valueType:"multi_enum"`,
select `multiple`) → `value`/`text` = **`[TechnicalName]`** (crochets car multiple ; PAS l'int), `operator:"="`,
`isKeysValues:"false"` :
```json
[{"entityPropertyName":"Status","operator":"=","propertyType":"","foreignEntityPropertyType":"",
  "isForeignKey":"false","isKeysValues":"false","foreignEntityPropertyName":"null","displayName":"Statut",
  "cultureCode":"","parameterName":"","valueType":"multi_enum","value":"[Ordered]","text":"[Ordered]"}]
```
- **`operator` = SYMBOLE** (`filters.js`) : `=` · `!` différent · `~` contient · `!~` · `!*` vide · `>` `>=` `<` `<=` · `><` entre.
- ⚠️ **`isKeysValues:"true"` réservé aux tree-data** (sinon force `value=text` → `Enum.Parse` plante, `RepositoryReflectionHelper.cs:549-560`). Valeurs/labels enum = `DynamicMultiEnumValues` (`Value` int, `TechnicalName`) lié au `DynamicMultiEnum` `Name=<Entité><Prop>` (ex. `BdgCommitmentStatus`).
- **Référence** : `isForeignKey:"true"`, `foreignEntityPropertyName`=prop affichée, `value`=Id/code cible.

**Widget « carte qui préfiltre »** (CodeRazor + CodeJavascript) :
```razor
@{ int nb = VP.Entities.GetCount("BdgCommitment", "EntityState == EntityState.Active && Status == 2"); }
<div id="qf2" class="kpi-card" style="cursor:pointer"><h2>@nb</h2><span>Commandé — cliquer pour filtrer</span></div>
```
```javascript
// SET (sans reload) — globaux dispo sur la page liste : postAsync, appBaseURL, $, resolveFilterBannerContainer,
// Handlebars, operatorLabels, ReplaceAsciiCodes, filterTypeToIcon, updateFiltersBanner, jErrorBox
(function(){var el=document.getElementById('qf2');if(!el)return;el.addEventListener('click',function(){
  var en='BdgCommitment',vn='';
  var f=[{entityPropertyName:'Status',operator:'=',propertyType:'',foreignEntityPropertyType:'',isForeignKey:'false',
    isKeysValues:'false',foreignEntityPropertyName:'null',displayName:'Statut',cultureCode:'',parameterName:'',
    valueType:'multi_enum',value:'[Ordered]',text:'[Ordered]'}];
  postAsync(appBaseURL+'/Api/DynamicFilters/SetFiltersSession',{EntityName:en,ViewName:vn,JsonFilters:JSON.stringify(f)})
   .then(function(){
     $('.dynamicListContainer').jtable('load');                       // relit la session (pas d'arg)
     var fbc=resolveFilterBannerContainer(document.querySelector('.dynamicFiltersModal'));
     var b=fbc.querySelector('[entity-filters-banner]');
     var tm=Handlebars.compile(document.getElementById('table-filter-banner-badge').innerHTML);
     var tg=fbc.querySelector('[table-filters-banner-tags]');tg.innerHTML='';
     var sn=fbc.querySelector('[selectedDynamicFiltersNumber]');if(sn){sn.textContent=String(f.length);sn.setAttribute('value',sn.textContent);}
     f.forEach(function(fo){var t=fo.text;if(t.charAt(0)==='['&&t.slice(-1)===']')t=t.slice(1,-1);
       var ol=operatorLabels[fo.operator]||fo.operator,tv=ReplaceAsciiCodes(t).split(',').map(function(v){return v.trim();}).join(', ');
       tg.insertAdjacentHTML('beforeend',tm({name:fo.displayName,propertyName:fo.entityPropertyName,entityName:en,viewName:vn,operator:ol,tooltip:fo.displayName+' '+ol+' '+tv,value:tv,icon:filterTypeToIcon(fo.valueType)}));});
     b.classList.remove('d-none');                                    // ⚠️ remove d-none, JAMAIS add d-flex
     updateFiltersBanner(fbc.querySelector('[table-filters-banner-container]'));
   }).catch(function(e){if(typeof jErrorBox==='function')jErrorBox((e&&e.ErrorMessage)||'Erreur',e);});});})();
```
- **Carte « Tous » (reset, sans reload)** : `ResetTableFilterSession` + `jtable('load')` + vider `[table-filters-banner-tags]` + compteur=`0` + **`bannerEl.classList.add('d-none')`** + `updateFiltersBanner(...)`.
- `postAsync`, `appBaseURL`, `jErrorBox`, `$` sont **globaux** sur la page liste (widget chargé via `/Users/Page?id=`).
- ⚠️ **Chaque widget = son propre `id`** + JS scopé (chaque widget rend `<style>`+`<div>`+`<script>` indépendamment).
- Le **compteur KPI Razor** (`GetCount`, l'INT marche en C#) reste le TOTAL par statut — inchangé par le filtre actif.
- ⚠️ `SetFiltersSession` **REMPLACE** tous les filtres de l'entité/vue (un quickfilter écrase les autres). Pour cumuler, lire+fusionner les filtres existants avant.
- Tester le rendu du widget sans navigateur : **`McpRenderDynamicPage`**.

> **2 formats de valeur enum (selon le chemin)** : via **`SetFiltersSession`** (chemin natif, recommandé) → `value`=**TechnicalName** `"[Ordered]"`. Via un `jtable("load",{jsonFilters})` **DIRECT** (`/Agl/GetDynamicEntitiesByFilter` → `RepositoryReflectionHelper.CreateMultiLambaExpressionBase`) → l'**INT** `"2"` marche aussi. Le serveur lit la clé **`type`** (≠ `valueType`, `FilterQuery.cs`).

### 1.5 FILTRES de liste (rendre des colonnes cherchables dans le bandeau) ✅

Une colonne devient filtrable dans le bandeau ⟺ **`DynamicFieldRole.Filter == true`** (par champ/rôle ;
`DynamicFieldRole.cs:53`, consommé par `DynamicFieldHelper.cs:608`). **PAS automatique** depuis l'affichage `Table`.
→ **`McpSetFieldsFilter(entityName, moduleId, "Champ1,Champ2,…", "true")`** — CodeUsing `using VPSoft.Domain.Enums;`
+ `using VPSoft.Utils.Helpers;` ; calqué sur `McpSetFieldsEditInLine` (pose `dfr.Filter` sur la liste blanche pour le
rôle de l'utilisateur courant, crée la `DynamicFieldRole` avec `Table=Display` si absente). **Config pure** (F5, pas de build).
Préfiltre permanent d'une vue : `DefaultFilters` (string JSON, même forme qu'en §1.4) sur la vue, lu par `DynamicService`
et compilé via `CreateMultiLambaExpression`.

---

## 2. CONFIDENTIALITÉ (onglet table + enregistrements)

### 2.1 Modèle (verbatim)

- **Niveau TABLE** (`DynamicSettings.cs:197-209`) : `IsConfidentialMaster (bool)` +
  `ConfidentialMasterName (string)` (mode esclave : suit l'entité maître) +
  `DynamicFieldConfidentialUsers : ISet<ConfidentialUser>` / `DynamicFieldConfidentialRoles :
  ISet<ConfidentialRole>` (config de l'onglet, avec `IsDisabled` par entrée).
- **Niveau ENREGISTREMENT** (`EntityDynamicAudit.cs:10-56`, hérité par TOUTE entité dynamique) :
  **`IsConfidential (bool)`**, **`ConfidentialUsers : ISet<User>`**, **`ConfidentialRoles :
  ISet<RoleInModule>`**.

### 2.2 Effet (verbatim — qui voit quoi)

`ConfidentialityHelper.JoinWhereConfidentiality(entityName)` (`VPSoft.Utils/Helpers/ConfidentialityHelper.cs:22-119`) :
**`IsConfidential == false OU (IsConfidential == true ET (user ∈ ConfidentialUsers OU rôle ∈
ConfidentialRoles))`** (+ suivi du maître si `ConfidentialMasterName`). Appliqué par
`VP.GetGlobalFilterForEntity` (`VP.cs:2072`) quand `filterQuery` contient
`FilterQueryConfidentiality` **ou `All`** (= le défaut de l'API REST/MCP : ✅ vérifié, un
enregistrement confidentiel reste visible à l'appelant si son rôle est listé). Accès unitaire :
`ConfidentialityHelper.CheckUserAccess(entityName, entityId)` (`:298-324`).

### 2.3 ✅ Fonctions MCP

**`McpSetTableConfidentiality`** — params `string entityName, string isConfidentialMaster,
string confidentialMasterName` · pose les 2 propriétés sur le `DynamicSettings` maître
(repository Edit, config → F5). ✅ testé set true → relecture → restore false sur `McpDemoArbre`.

**`McpSetRecordConfidentiality`** — params `string entityName, string recordCode, string isConfidential,
string userNamesCsv (UserName ou Email), string roleCodesCsv (RoleInModule.Code)` ·
**CodeUsing : `using NHibernate; using NHibernate.Criterion;`** — NHibernate direct (pattern
`McpSetReference`) :

```csharp
try {
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    var session = NHSessionHelper.GetCurrentSession();
    var crit = session.CreateCriteria(entityName); crit.Add(Restrictions.Eq("Code", recordCode));
    dynamic entity = crit.UniqueResult();
    if (entity == null) return "ERR enregistrement introuvable: " + entityName + " Code=" + recordCode;
    entity.IsConfidential = isConfidential == "true";
    entity.ConfidentialUsers.Clear(); entity.ConfidentialRoles.Clear();
    int nbU = 0; int nbR = 0;
    if (!string.IsNullOrEmpty(userNamesCsv)) {
        var names = userNamesCsv.Split(',').Select(x => x.Trim()).Where(x => x != "").ToList();
        var users = rm.UserRepository.GetMany(x => names.Contains(x.UserName) || names.Contains(x.Email)).ToList();
        foreach (var u in users) { entity.ConfidentialUsers.Add(u); nbU++; }
    }
    if (!string.IsNullOrEmpty(roleCodesCsv)) {
        var codes = roleCodesCsv.Split(',').Select(x => x.Trim()).Where(x => x != "").ToList();
        var roles = rm.RoleInModuleRepository.GetMany(x => codes.Contains(x.Code)).ToList();
        foreach (var r in roles) { entity.ConfidentialRoles.Add(r); nbR++; }
    }
    session.Update(entity); session.Flush();
    return JsonConvert.SerializeObject(new { success = true, record = recordCode, isConfidential = isConfidential == "true", users = nbU, roles = nbR });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé (démo)** : record `ZZ-CONF-1` → `IsConfidential=true` + rôle VPW Admin → toujours visible
via MCP (rôle autorisé) → supprimé. ⚠️ **Ne JAMAIS poser `isConfidential=true` sans users/roles**
incluant l'appelant : l'enregistrement deviendrait invisible (y compris via MCP).

---

## 3. PDF (onglet — modèles d'export PDF d'un enregistrement)

### 3.1 Modèle (verbatim)

**`EntityPDFModel`** (`VPSoft.Domain/Models/Entities/EntityPDFModel.cs:1-59`) :
`{ Name, SelectedModel (bool, modèle par défaut), EntityId (= DynamicSettings.Id), EntityName,
Sections : ISet<SectionConfigPDF>, PdfPageOrientation (enum EvoPdf : Portrait/Landscape) }`
**`SectionConfigPDF`** (`VPSoft.Domain/Models/Forms/SectionConfigPDF.cs:1-71`) :
`{ Name, SectionOrder, SectionContent (Razor/HTML), Status (bool), SectionType
(Section=0, PDFModel1=1, PDFModel2=2), EntityName }` — les sections `SectionType=Section` sont
réutilisables entre modèles d'une même entité.
**Génération** : contenu Razor des sections compilé → HTML → **EvoPdf**
(`EvoHtmlToPdfHelper.GetPdfBytesFromHtmlString`, `VPSoft.Utils/Helpers/EvoHtmlToPdfHelper.cs:22` —
A4, marges 0/16/0/16, JS activé). Génération à la demande aussi possible : `VP.Files.GetPDFFromHtml`.

### 3.2 ✅ Fonction MCP

**`McpCreatePdfModel`** — params `string entityName, string name, string sectionsJson, string orientation,
string selected` · **CodeUsing : `using EvoPdf;`** (l'enum `PdfPageOrientation` n'est pas dans les
usings par défaut) · `sectionsJson` = `[{"name":"Entête","order":1,"content":"<h1>…</h1>"}, …]`

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var dsId = sm.DynamicSettingsService.GetSingleSelect(x => x.EntityName == entityName && x.ViewName == null, x => x.Id);
    if (dsId == Guid.Empty) return "ERR DynamicSettings introuvable: " + entityName;
    var model = new EntityPDFModel();
    model.Name = name; model.EntityName = entityName; model.EntityId = dsId;
    model.SelectedModel = selected == "true";
    model.PdfPageOrientation = orientation == "Landscape" ? PdfPageOrientation.Landscape : PdfPageOrientation.Portrait;
    model.Sections = new HashSet<SectionConfigPDF>();
    var arr = JArray.Parse(sectionsJson); int order = 1;
    foreach (var it in arr) {
        var s = new SectionConfigPDF();
        s.Name = (string)(it["name"] ?? ("Section " + order));
        s.SectionOrder = it["order"] != null ? (int)it["order"] : order;
        s.SectionContent = (string)(it["content"] ?? "");
        s.Status = true; s.SectionType = SectionConfigPDFType.Section; s.EntityName = entityName;
        model.Sections.Add(s); order++;
    }
    bool ok = sm.EntityPDFModelService.Create(model);
    return JsonConvert.SerializeObject(new { success = ok, pdfModelId = model.Id, name = name, sections = model.Sections.Count, orientation = model.PdfPageOrientation.ToString() });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé (démo)** : `ZZ Fiche véhicule (exemple MCP)` (2 sections) créé sur `McpDemoVehicule`,
vérifié en base — visible dans l'onglet PDF.

---

## 4. ALERTES (onglet — `DataAlert` : notification/mail sur événement ou batch)

### 4.1 Modèle (verbatim `VPSoft.Domain/Models/Entities/DataAlert.cs:1-122`)

`DataAlert { Code, AlertedEntity, DataAlertType (Mail=0, Notification=1, MailAndNotification=2),
DataAlertMailType (Simple/Bundled), **ActionType : AlertActions [Flags] (None=0, Create=1, Edit=2,
CreateAndEdit=3, Delete=4)** ✅ vérifié, DataAlertSourceType (Standard=0, Workflow=1, Batch=2),
EntityWorkflowAction? (si Workflow), UseBatch + Frequency + StartExecuteDate/LastExecuted (mode
planifié), DeactivatableAlert/Notif, CustomUrl, RolesToAlert : ISet<RoleDataAlert{Role}>, FieldsToAlert :
ISet<FieldDataAlert{Field = propriété User de l'entité, ex. "Responsable"}>, EntityProperties :
ISet<EntityPropertyDataAlert> (conditions), AlertRule (BusinessRule Type=8), Attachments }`
+ contenus localisés via `CultureParameter` : `LocalizedName`, `LocalizedSubject`, `LocalizedMessage`,
`LocalizedNotificationTitle`, `LocalizedNotification`.

**Conditions** `EntityPropertyDataAlert` : `{ AlertedProperty, AlertedPropertyType (Enum=0, Int, Date,
String, Bool, Guid, Link, Question), UsedOperator (Equal=0, Different, Contain, GreaterThan[Equal],
LessThan[Equal], StartBy, Before, After, FromTo, In, NotIn, IfChange=13), ValueToCompare, DateShift(+Unit),
CompareWithOldValue/UseOldValue }`.

### 4.2 Pipeline (verbatim)

**`DataAlertListener`** (`VPSoft.Services/Listeners/DataAlertListener.cs:32-476`) — listener NHibernate
post-insert/update/delete : sélectionne les `DataAlert` **`EntityState==Active` + Standard + !UseBatch**
de l'entité, évalue l'**`AlertRule.ActivationRuleTreeId`** (arbre de condition), cible les destinataires
via `ActionRuleTreeId` (RuleTree sur **User**) puis filtre par `AffectationRuleTreeId`
(`MappedUsersForAlert` `:353-438`), envoie (mail/notification) et pose `LastExecuted`. Les alertes
**Workflow** sont déclenchées par la création d'`EntityWorkflowHistory` ; les **Batch/planifiées** par
le scheduler. → Une alerte **Inactive ne part JAMAIS** (filtre EntityState).

### 4.3 ✅ Fonction MCP

**`McpCreateDataAlert`** — params `string entityName, string code, string name, string actionType
(Create|Edit|CreateAndEdit|Delete), string subject, string message, string roleCodesCsv, string active`
· **CodeUsing : `using VPSoft.Domain.Enums.Notifications;`** ⚠️ (namespace des enums DataAlert*
ABSENT des usings par défaut — sinon `null` muet, vécu) :

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    if (sm.DataAlertService.GetMany(x => x.Code == code).Any()) return "ERR Code alerte deja existant: " + code;
    var alert = new DataAlert();
    alert.Code = code; alert.AlertedEntity = entityName;
    alert.DataAlertType = DataAlertType.Notification;
    alert.DataAlertMailType = DataAlertMailType.Simple;
    alert.ActionType = (AlertActions)Enum.Parse(typeof(AlertActions), actionType);
    alert.DataAlertSourceType = DataAlertSourceType.Standard;
    alert.DeactivatableAlert = true; alert.DeactivatableNotif = true; alert.UseBatch = false;
    alert.EntityState = active == "true" ? EntityState.Active : EntityState.Inactive;
    if (!string.IsNullOrEmpty(roleCodesCsv)) {
        var codes = roleCodesCsv.Split(',').Select(x => x.Trim()).Where(x => x != "").ToList();
        var roles = rm.RoleInModuleRepository.GetMany(x => codes.Contains(x.Code)).ToList();
        foreach (var r in roles) { alert.RolesToAlert.Add(new RoleDataAlert { Role = r, DataAlert = alert }); }
    }
    bool ok = sm.DataAlertService.Create(alert);
    Action<string, string> setCulture = (prop, val) => { if (string.IsNullOrEmpty(val)) return;
        var rc = new ResourceCultureJson { resourceKey = prop, resourceValues = new List<ResourceCultureValueJson> {
            new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = val },
            new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = val } } };
        sm.CultureParameterService.SaveOrUpdate(rc, alert.Id, typeof(DataAlert).GetProperty(prop)); };
    setCulture("LocalizedName", name); setCulture("LocalizedSubject", subject);
    setCulture("LocalizedMessage", message); setCulture("LocalizedNotificationTitle", subject);
    setCulture("LocalizedNotification", message);
    return JsonConvert.SerializeObject(new { success = ok, alertId = alert.Id, code = code, state = alert.EntityState.ToString(), roles = alert.RolesToAlert.Count });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé (démo)** : `ZZ_MCP_ALERT` créée **Inactive** sur `McpDemoVehicule` (rôle VPW Admin),
5 valeurs localisées vérifiées dans `CultureParameter`. **Standard : créer Inactive, activer
explicitement** (`EntityState=Active` via service Edit) quand la config est complète — une alerte
Active part à CHAQUE événement correspondant. Conditions fines : ajouter des
`EntityPropertyDataAlert` et/ou une `AlertRule` (cf. `NOCODE-…BusinessRules.md` — kind `Alert`,
`EntityId = DataAlert.Id`, arbres activation/action/affectation).

---

## 5. AIDES (onglet — aide en ligne contextuelle `HelpOnline`)

### 5.1 Modèle (verbatim `VPSoft.Domain/Models/Builder/HelpOnline.cs:14-58`)

`HelpOnline { EntityName, ViewName (null = table maître), RoleInModules : ISet<RoleInModule>,
LocalizedName, LocalizedDescriptionCreate / Edit / Detail / **List** (4 contextes, multilingues via
CultureParameters) }`. Affichage : bouton « aide » sur formulaire/liste si
`HelpOnlineService.HasHelpOnLineForUser(user, entityName, action, viewName)`
(`HelpOnlineService.cs:170-213`) — l'utilisateur doit avoir un rôle ∈ `RoleInModules` ET le contenu
du contexte non vide. ⚠️ **Quirk vérifié dans le code : la condition est `GetCountByFilter(…) == 1`** —
si DEUX aides matchent le même contexte/rôle, le bouton DISPARAÎT. Une seule aide par
entité+vue+contexte+rôle.

### 5.2 ✅ Fonction MCP

**`McpCreateOnlineHelp`** — params `string entityName, string title, string helpList, string helpCreate,
string helpEdit, string helpDetail, string roleCodesCsv, string viewName` · CodeUsing : *(aucun)* :

```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    var roleCodes = (roleCodesCsv ?? "").Split(',').Select(x => x.Trim()).Where(x => x != "").ToList();
    var roles = rm.RoleInModuleRepository.GetMany(x => roleCodes.Contains(x.Code)).ToList();
    if (roles.Count == 0) return "ERR aucun RoleInModule trouve pour: " + roleCodesCsv;
    var help = new HelpOnline();
    help.EntityName = entityName;
    help.ViewName = string.IsNullOrEmpty(viewName) ? null : viewName;
    help.RoleInModules = new HashSet<RoleInModule>(roles);
    rm.HelpOnlineRepository.Save(help);
    Action<string, string> setCulture = (prop, val) => { if (string.IsNullOrEmpty(val)) return;
        var rc = new ResourceCultureJson { resourceKey = prop, resourceValues = new List<ResourceCultureValueJson> {
            new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = val },
            new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = val } } };
        sm.CultureParameterService.SaveOrUpdate(rc, help.Id, typeof(HelpOnline).GetProperty(prop)); };
    setCulture("LocalizedName", title);
    setCulture("LocalizedDescriptionList", helpList); setCulture("LocalizedDescriptionCreate", helpCreate);
    setCulture("LocalizedDescriptionEdit", helpEdit); setCulture("LocalizedDescriptionDetail", helpDetail);
    return JsonConvert.SerializeObject(new { success = true, helpId = help.Id, entityName = entityName, roles = roles.Count });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

✅ **Testé (démo)** : « Aide - Suivi des véhicules » créée sur `McpDemoVehicule` (contenus Liste/
Create/Edit, rôle VPW Admin), relecture `LocalizedName` + `LocalizedDescriptionList` OK — visible
dans l'onglet « Aides » de l'AGL et via le bouton d'aide côté utilisateur.

---

## 6. Gotchas (vécu sur la démo)

- **`using VPSoft.Domain.Enums.Notifications;` OBLIGATOIRE** pour `DataAlertType`/`AlertActions`/
  `DataAlertSourceType`/`DataAlertMailType` (hors usings par défaut ⇒ `null` muet). Idem
  **`using EvoPdf;`** pour `PdfPageOrientation`.
- **`AlertActions` est un `[Flags]`** : Create=1, Edit=2, CreateAndEdit=3, Delete=4 (PAS 0/1/2).
- **`DataAlert.LocalizedName` n'est pas mappé** (propriété non-virtual) : `get_many_select` la lit
  `null` — lire le nom via `CultureParameter` (`ParameterId == <alertId>`).
- **Toujours créer les alertes `Inactive`** puis activer explicitement (une alerte Active part à
  chaque Create/Edit/Delete correspondant, y compris ceux des tests MCP).
- **Confidentialité : ne jamais poser `IsConfidential=true` sans s'inclure** (user ou rôle), sous
  peine de perdre l'accès à l'enregistrement (même via MCP, filterQuery `All` applique le filtre).
- **Aides : 1 seule par entité+vue+contexte+rôle** (la visibilité teste `count == 1`).
- **Visuels par RÔLE** : une section est liée à UN `RoleInModule` ; pour plusieurs profils, créer
  une section par rôle (mêmes colonnes). Le rendu ne montre que les sections des rôles de
  l'utilisateur courant.
- **Widget dans une card** : `Content` = l'**Id** du DynamicWidget (string) — `McpAddTableWidget`
  accepte aussi le **Code** et résout l'Id.
