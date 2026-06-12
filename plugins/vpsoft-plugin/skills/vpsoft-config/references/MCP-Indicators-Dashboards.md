# Indicateurs & Dashboards via MCP — fonctions créées et validées

> Méthode validée selon le contrat FormBuilder (cf. `MCP-DynamicFunctions-FormBuilder.md` dans ce même
> dossier). Fil conducteur : **workflow dynamique → indicateur → dashboard**.
> App : `demo-vesta-10-5` (`https://demo.vpwhite.com/Vesta_Demo`). Module Migration
> `f368e802-3893-437a-8c77-acbd44c89b1a`, dossier fonctions MCP `d019db8c-8b9a-4922-b3d2-d3145638b1b2`.
> Entités de démo : `McpDemoVehicule` (parent) / `McpDemoIntervention` (enfant, `Cout` decimal, ref `Vehicule`).

Toutes les fonctions suivent le contrat FormBuilder : **paramètres `string`**, `returnValue = System.Object`,
**try/catch interne** qui renvoie `ex.Message + GetType().FullName + inner + stack` (le wrapper avale les
exceptions → `null` sinon), résolution via `var sm = AppDependencyResolver.GetService<IServiceManager>();`.

---

## 1. `McpCreateEntityWorkflow` — workflow dynamique

**FunctionId** : `3a6030c9-7639-4558-9e6b-0c546f978422`
**Paramètres** : `string entityName, string name`

### Modèle & preuves source
- `VPSoft.Presentation/Controllers/EntityWorkflowController.cs:194-231` — chemin Create validé :
  `new EntityWorkflow { Name, EntityState, EntityName }` → `EntityWorkflowService.Create(wf)` puis
  **3×** `CreateEntityWorkflowDefautStatus(wf, EntityWorkflowStatusCategory.{ToBeSubmitted|Approved|Declined})`.
- `VPSoft.Services/Entities/EntityWorkflowService.cs:54` `Create()` = `Repository.Save()` ;
  `:171` `CreateEntityWorkflowDefautStatus()` génère `Code = {EntityName}-{Name}-{category.GetDescription()}`,
  `Order = (int)category + 1`, + un `CultureParameter` par culture active.
- `VPSoft.Domain/Models/Workflow/EntityWorkflow.cs` — `Name`, `EntityName`, `EntityState`.

### Logique
Validation → contrôle d'existence (`GetManySelect` sur `EntityName`) →
`new EntityWorkflow { Name, EntityName, EntityState = EntityState.Active }` → `Create` →
3 appels `CreateEntityWorkflowDefautStatus`. **Usings** : `VPSoft.Domain.Models.Workflow`, `VPSoft.Domain.Enums`.

### Test & résultat validé
`McpCreateEntityWorkflow("McpDemoVehicule", "Cycle de vie")` →
`{ success: true, workflowId: "dd4f3cd7-080b-4d29-9dc4-3fa6996ba016" }`.

Vérification en base (`EntityWorkflowStatus`, `EntityName == "McpDemoVehicule"`) — **3 états créés** :

| Code | LocalizedName | Order |
|------|---------------|-------|
| McpDemoVehicule-Cycle de vie-… | À soumettre | 1 |
| McpDemoVehicule-Cycle de vie-… | Approuvé | 2 |
| McpDemoVehicule-Cycle de vie-… | Refusé | 3 |

---

## 2. `McpCreateIndicator` — indicateur agrégé

**FunctionId** : `2e036c63-0032-47be-a987-9a2dc3526630`
**Paramètres** : `string entityName, string localizedName, string measurePropertyName, string aggregation,`
`string groupEntityName, string groupPropertyName, string groupTreePath`

### Modèle & preuves source
- `VPSoft.Services/Indicators/IndicatorService.cs:1572` `CreateIndicator(IndicatorPreviewFormModel)` →
  utilise `.Indicator` (sinon `null`) → `new Indicator()` → `ApplyFormModelToIndicator` → `Create` →
  `SaveAllCultureParameters`.
- `:1632` `ApplyFormModelToIndicator` : `EntityName = EntityNameRelated = formModel.EntityName` ;
  `DefaultChartRepresentation ?? GridMatrice` ; `IsDisplayTotals ?? None` ;
  `OriginType = IsTemplate ? Standard : User` ; appelle `ApplyCalculatedData` + `ApplyIndicatorFilters`.
- `:1656` `ApplyCalculatedData` → `IndicatorData` : `EntityPropertyName = PropertyName`,
  `SqlAggregation = CalculationType`, `IndicatorDataFormat = DataFormat`, `Round = DecimalDigits`, `TreePath = Path`.
- `:1732` `ApplyIndicatorFilters` → `IndicatorFilter` : `EntityPropertyDisplay = PropertyName`,
  `TreePath = Path`, `IndicatorFilterFormat = FilterFormat`, `SortingOrder = SqlOrderBy`,
  **`FilterType = FilterType.Aggregate`**.
- Contrats : `VPSoft.Domain/Contracts/Indicators/Configurations/IndicatorOverrideFormModel.cs`
  (`IndicatorOverrideFormModel`, `CalculatedDataOverrideFormModel`, `AggregatedDataOverrideFormModel`).
- `VPSoft.Domain/Utils/Database/SqlTools.cs:412` `enum SqlAggregation { count, sum, average, minimum, maximum, None }`
  — **membres en minuscules** (`Enum.TryParse(..., true, ...)`).

### Logique
`Enum.TryParse<SqlAggregation>(aggregation, true)` →
`CalculatedDataOverrideFormModel { Type=Default, EntityName, PropertyName=measure, Path=entityName,`
`CalculationType=agg, DataFormat=Currency, DecimalDigits=2 }` (mesure) +
`AggregatedDataOverrideFormModel { EntityName=group, PropertyName=groupProp, Path=groupTreePath,`
`FilterFormat=None }` (dimension) → `IndicatorOverrideFormModel { ChartRepresentation=Bar,`
`IsDisplayTotals=Vertical, IsTemplate=true, CalculatedData, AggregatedData }` →
`IndicatorPreviewFormModel { Indicator, IsPreview=false }` → `CreateIndicator`.
**Pattern regroupement par référence** : `Path/TreePath = "EntiteRacine.ProprieteReference"`.

### Test & résultat validé
`McpCreateIndicator("McpDemoIntervention", "Coût total par véhicule", "Cout", "sum",`
`"McpDemoVehicule", "Immatriculation", "McpDemoIntervention.Vehicule")` →
`{ success: true, indicatorId: "c95f655a-ad63-4150-b311-3d5ba20964e2" }`.

Vérification en base :
- `Indicator` : `EntityName = McpDemoIntervention`, `DefaultChartRepresentation = 1` (Bar),
  `IsDisplayTotals = 3` (Vertical), `OriginType = 0` (Standard).
- `IndicatorData` : `SqlAggregation = sum`, `EntityPropertyName = Cout`, `IndicatorDataFormat = 1`
  (Currency), `TreePath = McpDemoIntervention`, `Round = 2`.
- `IndicatorFilter` : `EntityName = McpDemoVehicule`, `EntityPropertyDisplay = Immatriculation`,
  `TreePath = McpDemoIntervention.Vehicule`, `FilterType = 0` (Aggregate).

---

## 3. `McpCreateDashboard` — conteneur module + dashboard à widgets

**FunctionId** : `b173685e-20ce-4353-bd90-0dd5aa31d85d`
**Paramètres** : `string moduleId, string code, string name`

### Modèle & preuves source
- `VPSoft.Services/UI/Dashboards/DynamicDashboardService.cs:133-195` `CreateDynamicDashboard(DynamicDashboardCreateViewModel)` :
  résout le dossier racine du module (`ModuleId`) ou `FolderId`, **crée automatiquement une `DashCollection`**
  (`Name = Code`, `:154-157`), ajoute le `RoleInModule` de l'utilisateur courant, sauvegarde, écrit le
  libellé via `CreateResourceCultureEntityProperty`. Ne retourne que `dynamicDashboard.Id`.
- `:89` `GetSingle(Guid id, params fetches)` — permet `GetSingle(id, x => x.DashCollections)` pour
  récupérer la `DashCollection` auto-créée. `DynamicDashboard.cs:36` `ISet<DashCollection> DashCollections`.
- `VPSoft.Services/UI/Dashboards/DashboardService.cs:485-541` `CreateDashboard(DashDashboardFormModel)` :
  `new Dashboard()` → `Create` → libellé via `CultureParameterService.SaveOrUpdate({ResourceCulture},`
  `"tab_title", dashboard.Id, ParameterName.DashboardTitle)` → si `DashboardCollectionId` fourni, crée un
  **`DashboardUser`** liant Dashboard ↔ DashCollection (`IsAdminDashboard` ou user courant).
- `VPSoft.Domain/Contracts/Dashboards/DynamicDashboardCreateViewModel.cs` (`Code` requis, `Name`
  `ResourceCultureJson`, `FolderId?`/`ModuleId?`) ; `DashboardCollectionFormModel.cs:22-53` `DashDashboardFormModel`.
- `CultureParameterService.cs:341-352` `CreateResourceCultureEntityProperty` fait
  `.First(x => x.cultureCode == cultureCourante)` → fournir **une valeur par culture active** ;
  `:542-547` `SaveOrUpdate(..., "tab_title", ...)` matche `resourceKey == "tab_title"`.
- Cultures actives (requête `Culture`) : `fr-FR` (défaut) + `en-GB`.

### Logique
Validation + contrôle d'unicité (`DynamicDashboardService.GetCountByFilter(x => x.Code == code)`) →
`DynamicDashboardCreateViewModel { Code, Name (resourceValues fr-FR + en-GB), ModuleId }` →
`CreateDynamicDashboard` → `GetSingle(id, x => x.DashCollections)` → `DashCollections.First().Id` →
`DashDashboardFormModel { DashboardCollectionId, ResourceCulture (resourceKey="tab_title"),`
`OrderBy=0, IsAdminDashboard=true }` → `CreateDashboard`. Renvoie `dynamicDashboardId`, `dashCollectionId`,
`dashboardId`. **Usings** : `VPSoft.Domain.Contracts.Dashboards`, `VPSoft.Domain.Contracts.App`,
`VPSoft.Domain.Models.UI.Dashboards`.

### Test & résultat validé
`McpCreateDashboard("f368e802-3893-437a-8c77-acbd44c89b1a", "MCP_DASH_VEHICULE", "Suivi Véhicules MCP")` →
`{ success: true, dynamicDashboardId: "2ba1ed52-aaee-4b1f-9fdd-40ee49f41d1a",`
`dashCollectionId: "080358e4-a714-4e9d-96b7-3f83202d04bc", dashboardId: "bdfa3383-816b-4f5d-9cd4-1d49f5fa46d7" }`.

Vérification en base :
- `DynamicDashboard` : `Code = MCP_DASH_VEHICULE`, module = `Migration`.
- `DashboardUser` `8eacf74f-…` : `Dashboard = bdfa3383`, `DashCollection = 080358e4` (Name MCP_DASH_VEHICULE),
  `IsAdminDashboard = true`, `OrderBy = 0`.

---

## 4. `McpAddIndicatorWidget` — widget indicateur (ReportCST)

**FunctionId** : `a6d9cb5e-4a0e-4a0b-a58a-3a42bb215890`
**Paramètres** : `string dashboardId, string indicatorId, string title, string x, string y, string w, string h`

### Modèle & preuves source
- `VPSoft.Services/UI/Dashboards/DashboardService.cs:676-735` `CreateWidgetConfig(WidgetFormModel)` :
  case **`ReportCST`** (`:686-691`) → `new DashboardReportWidget()` dont
  `Indicator = IndicatorRepository.LoadReference(((ReportWidgetDataFormModel)Data).IndicatorUserId.Value)`.
  → **`IndicatorUserId` = l'`Indicator.Id`** (pas un IndicatorUser). Position/taille depuis le form model,
  `Save`, puis libellé via `CultureParameterService.SaveOrUpdate({ResourceCulture}, widget.Id, nameof(DashboardWidget))`.
- `VPSoft.Domain/Contracts/Dashboards/DashboardCollectionFormModel.cs:82-134` `WidgetFormModel`
  (`DashboardId`, `WidgetType`, `XPosition/YPosition/Height/Width`, `Data`, `ResourceCulture`) +
  `ReportWidgetDataFormModel { Guid? IndicatorUserId }`.
- `VPSoft.Domain/Models/UI/Dashboards/DashWidget.cs:123-124` `DashWidgetType.ReportCST = 8`.
- `CultureParameterService.cs:887-896` `SaveOrUpdate(IEnumerable<ResourceCultureJson>, Guid, string)` itère
  sans contrôle de null → fournir un `ResourceCulture` non null avec `resourceKey`. Ligne 894 :
  **`PropertyName = resourceCultureJson.resourceKey`** (la clé devient le `PropertyName` du `CultureParameter`).

### ⚠️ Bug du nom affiché en GUID — corrigé
Le chemin de **lecture** du libellé est `DashboardService.cs:1312-1389` `MapWidgetDtoToViewModel` :
`LocalizedName = CultureParameterHelper.GetLocalizedValue(widgetDto.CultureParameters`
`.Where(p => p.PropertyName == DashboardWidget.LocalizedName), DashboardWidget.LocalizedName)` —
or `DashboardWidget.LocalizedName == "LocalizedName"` (`DashboardWidget.cs:12`).
La 1ʳᵉ version utilisait `resourceKey = "name"` → `PropertyName = "name"` → **aucun match** côté lecture →
libellé vide → l'UI **retombe sur le GUID du widget**. **Correctif** : `resourceKey = "LocalizedName"`.
> Mise à jour du code via `McpUpdateFunctionCode` (cf. §6, le PATCH générique sur `DynamicFunction` renvoie 500).

### Logique
Validation → parse positions (défauts 0/0/8/5) → `ResourceCultureJson { resourceKey="LocalizedName",`
`resourceValues fr-FR + en-GB = title }` → `WidgetFormModel { DashboardId, WidgetType=ReportCST, X/Y/W/H,`
`ResourceCulture, Data = ReportWidgetDataFormModel { IndicatorUserId = Guid.Parse(indicatorId) } }` →
`CreateWidgetConfig`. **Usings** identiques à `McpCreateDashboard`.

### Test & résultat validé
`McpAddIndicatorWidget("bdfa3383-816b-4f5d-9cd4-1d49f5fa46d7", "c95f655a-ad63-4150-b311-3d5ba20964e2",`
`"Coût par véhicule", "0", "0", "8", "6")` →
`{ success: true, widgetId: "97adb7b9-44ac-43dd-8683-b45e009384c0", … }`.

Vérification en base (`DashboardReportWidget`, `Dashboard.Id == bdfa3383`) :
- `Id = 97adb7b9-…`, `PositionX=0, PositionY=0, Width=8, Height=6`,
- `Indicator.Id = c95f655a-…`, `Indicator.EntityName = McpDemoIntervention`,
  `Indicator.DefaultChartRepresentation = 1` (Bar).

---

## 5. `McpCreateMultiMeasureIndicator` — indicateur report à plusieurs mesures

**FunctionId** : `383a2696-0b96-4b76-99ba-511a6ba4631c`
**Paramètres** : `string entityName, string localizedName, string measuresJson,`
`string groupEntityName, string groupPropertyName, string groupTreePath`

### Pourquoi « plusieurs indicateurs » = plusieurs mesures
`DashboardReportWidget : DashboardWidget` (`DashboardWidget.cs:32-35`) porte **un seul `Indicator`**.
« Un report avec plusieurs indicateurs » se modélise donc par **un `Indicator` à plusieurs mesures** :
`Indicator.IndicatorDataList = ISet<IndicatorDataBase>` (`Indicator.cs`), chaque `IndicatorData`
(`IndicatorData.cs`, `IndicatorDataBase` : `EntityPropertyName`, `SqlAggregation`, `IndicatorDataFormat`,
`Round`, `TreePath`) = **une série/mesure**. Côté form model, chaque mesure = un
`CalculatedDataOverrideFormModel` dans `IndicatorOverrideFormModel.CalculatedData` (List).

### Logique
`measuresJson` est un **tableau JSON** d'objets `{ property, agg, name, format, decimals }` (parse `JArray`).
Pour chaque entrée → un `CalculatedDataOverrideFormModel { Type=Default, EntityName=entityName,`
`PropertyName=property, Path=entityName, LocalizedName=name, CalculationType=Enum.Parse<SqlAggregation>(agg,true),`
`DataFormat=Enum.Parse<IndicatorDataFormat>(format,true), DecimalDigits=decimals }`. Une seule dimension
`AggregatedDataOverrideFormModel { EntityName=group, PropertyName=groupProp, Path=groupTreePath, FilterFormat=None }`.
→ `IndicatorOverrideFormModel { ChartRepresentation=Bar, IsDisplayTotals=Vertical, IsTemplate=true,`
`CalculatedData (N), AggregatedData (1) }` → `IndicatorPreviewFormModel { Indicator, IsPreview=false }` →
`IndicatorService.CreateIndicator`. Renvoie `indicatorId` + `measureCount`.
**Usings** : `VPSoft.Domain.Contracts.Indicators.Configurations`, `VPSoft.Domain.Utils.Database`,
`VPSoft.Domain.Enums`, `Newtonsoft.Json.Linq`.

### Test & résultat validé
```
McpCreateMultiMeasureIndicator(
  "McpDemoIntervention", "Coûts par véhicule (multi-mesures)",
  "[{\"property\":\"Cout\",\"agg\":\"sum\",\"name\":\"Total\",\"format\":\"Currency\",\"decimals\":2},
    {\"property\":\"Cout\",\"agg\":\"average\",\"name\":\"Moyenne\",\"format\":\"Currency\",\"decimals\":2},
    {\"property\":\"Cout\",\"agg\":\"count\",\"name\":\"Nb interventions\",\"format\":\"Number\",\"decimals\":0}]",
  "McpDemoVehicule", "Immatriculation", "McpDemoIntervention.Vehicule")
→ { success: true, indicatorId: "7fdfdec6-1d15-4ce4-bba1-4df5530008c3", measureCount: 3 }
```
Vérification en base (`IndicatorData`, `Indicator.Id == 7fdfdec6`) — **3 mesures** :

| SqlAggregation | EntityPropertyName | IndicatorDataFormat |
|----------------|--------------------|---------------------|
| 1 (sum)     | Cout | 1 (Currency) |
| 2 (average) | Cout | 1 (Currency) |
| 0 (count)   | Cout | 3 (Number)   |

Puis `McpAddIndicatorWidget(...)` (version corrigée) → widget `c681cb1e-ed77-4a24-a128-b45e00a25abf`.
`CultureParameter` du widget : `PropertyName = "LocalizedName"`, `EntityName = "DashboardWidget"`,
`Name = "Coûts par véhicule (multi-mesures)"` (fr-FR + en-GB) → **nom correctement affiché** (plus de GUID).

---

## 6. `McpUpdateFunctionCode` — mise à jour du code d'une DynamicFunction

**FunctionId** : `2f28143c-f5e3-40cf-aff7-a727e68cb608`
**Paramètres** : `string functionId, string codeFunction, string codeUsing, string parameters`

### Pourquoi
Le PATCH générique (`update_entity_patch` MCP) sur `DynamicFunction` renvoie **500**. Cette fonction
recharge la `DynamicFunction` et appelle `DynamicFunctionService.Edit` (`:138`) — le bon chemin métier.

### Logique
`DynamicFunctionService.GetSingle(Guid.Parse(functionId))` (`:153`) → met à jour `CodeFunction`,
et si non vides `CodeUsing` / `Parameters` → `Edit(fn)`. Renvoie `success` + `functionName`.
**Usings** : `VPSoft.Domain.Models.Builder`.
> Utilisée pour appliquer le correctif `resourceKey "name" → "LocalizedName"` de `McpAddIndicatorWidget`.

---

## 7. `McpCreateTypedIndicator` — indicateur de n'importe quel type de graphe

**FunctionId** : `11c54e35-846d-4984-ada8-9c20cc41ea9f`
**Paramètres** : `string entityName, string localizedName, string measuresJson, string chartType,`
`string groupEntityName, string groupPropertyName, string groupTreePath`

### Modèle & preuves source
- `VPSoft.Domain/Models/Indicators/Indicator.cs:149-206` `enum ChartRepresentation` — **20 types exploitables** :
  `Grid=0, Bar=1, HorizontalBar=2, Lines=3, Area=4, Pie=5, Radar=6, StackedBar=7, HorizontalStackedBar=8,`
  `PolarArea=9, BarSideBySide=10, HorizontalBarSideBySide=11, AreaSideBySide=14, LinesSideBySide=15,`
  `GridMatrice=16, Card=17, StackedBar100=18, HorizontalStackedBar100=19, Doughnut=20, HalfDoughnut=21`.
  Le type est porté par `Indicator.DefaultChartRepresentation` (`:46`).
- `IndicatorDataFormat` (`:263-308`) : `None=0, Currency=1, Decimal=2, Number=3, Percentage=4, Custom=5,`
  `YearDate=6, MonthYearDate=7, StandardDate=8, StripHtml=9, Truncate=10`.

### Logique
Identique à `McpCreateMultiMeasureIndicator` (§5) **+ un paramètre `chartType`** :
`Enum.TryParse<ChartRepresentation>(chartType, true, out chart)` (défaut `Bar` si vide/invalide) →
`IndicatorOverrideFormModel.ChartRepresentation = chart`. Mesures via `measuresJson` (1..N), dimension
de regroupement optionnelle (laisser `groupEntityName`/`groupPropertyName` vides ⇒ KPI simple, ex. `Card`).

### Test & résultats validés
Création d'**un indicateur par type** (SUM(Cout) par véhicule ; SUM+AVG+COUNT pour les types stacked/side-by-side)
puis ajout sur le dashboard dédié `MCP_DASH_ALLTYPES` (`McpAddIndicatorWidget`, version corrigée).
Vérifications en base :
- `DashboardReportWidget` du dashboard `9a0039c9` : **20 widgets**.
- `Indicator.DefaultChartRepresentation` conforme (échantillon) : Grid=0, Radar=6, Card=17,
  HorizontalStackedBar100=19, HalfDoughnut=21.
- `CultureParameter` des widgets : `PropertyName="LocalizedName"`, fr-FR + en-GB renseignés
  (nom affiché correct, plus de GUID).

| # | ChartRepresentation | Mesures | IndicatorId |
|---|---------------------|---------|-------------|
| 0 | Grid → **GridMatrice** (cf. §10) | 1 | `0cbe03f4-34ef-415a-8113-294705035377` |
| 1 | Bar | 1 | `9d02ed8a-c1c5-4ad0-afe3-ab7022e51ecd` |
| 2 | HorizontalBar | 1 | `9fd152d7-9c86-441e-8787-04a499801a5a` |
| 3 | Lines | 1 | `dfd9f4ba-5845-49bd-9732-249bc126788c` |
| 4 | Area | 1 | `b62611e6-6336-4731-80c9-f0f275593a75` |
| 5 | Pie | 1 | `a3d660bc-9c6a-4454-901d-2573795a20af` |
| 6 | Radar | 1 | `3ab4734f-9a73-4789-a1cf-913eee1d3e97` |
| 7 | PolarArea | 1 | `dcff0214-3292-48ed-b538-710b26ac4701` |
| 8 | GridMatrice | 1 | `e54858df-0532-4e47-8841-3eb4cdaaa12e` |
| 9 | Card | 1 (sans regroupement) | `f81fe5c7-e877-47d6-89fa-bb2dcc86ff2c` |
| 10 | Doughnut | 1 | `c979e0df-cba8-46ff-ab19-37cdee399901` |
| 11 | HalfDoughnut | 1 | `567130e5-1b5b-48fd-9409-a380bfa1573f` |
| 12 | StackedBar | 3 | `0b60b410-df83-41a3-a8f3-d5cc6c3a22ed` |
| 13 | HorizontalStackedBar | 3 | `cfd61789-b46a-412f-8d3a-0f399af4eb0f` |
| 14 | BarSideBySide | 3 | `fc0e71e5-e811-432d-ae82-23615b431ddc` |
| 15 | HorizontalBarSideBySide | 3 | `25da407e-bf65-4ea9-bd7f-d367ef04186c` |
| 16 | AreaSideBySide | 3 | `9d60bc91-9263-4d8a-b51c-45b4b2efb9b5` |
| 17 | LinesSideBySide | 3 | `e5059664-9600-44c0-a4b6-4fbebd2cc4b1` |
| 18 | StackedBar100% | 3 | `b71364db-9b36-4dca-b640-84481e7f7333` |
| 19 | HorizontalStackedBar100% | 3 | `2b53cf7d-41e5-44ce-8b4b-316864fc7213` |

> Dashboard « tous les types » : `MCP_DASH_ALLTYPES` — DynamicDashboard `e989498b-4a0f-4da1-8398-442a14198a53`,
> DashCollection `f975c2d6-21ea-4727-81ab-fc1024bc383a`, Dashboard `9a0039c9-7ef1-4800-9b24-3e866a7416b7`
> (grille 3 colonnes, widgets 4×5).

---

## 8. Chaîne complète validée

```
DynamicDashboard (2ba1ed52, module Migration, Code MCP_DASH_VEHICULE)
  └─ DashCollection (080358e4, Name MCP_DASH_VEHICULE)
       └─ DashboardUser (8eacf74f, IsAdminDashboard=true)
            └─ Dashboard (bdfa3383)
                 └─ DashboardReportWidget (97adb7b9, 8×6)
                      └─ Indicator (c95f655a — SUM(Cout) par McpDemoVehicule.Immatriculation, Bar)
                           ├─ IndicatorData  (sum / Cout / Currency / TreePath McpDemoIntervention)
                           └─ IndicatorFilter (McpDemoVehicule.Immatriculation / Aggregate /
                                               TreePath McpDemoIntervention.Vehicule)
```

Fil « workflow → indicateur → dashboard » bouclé. **Aucun rebuild requis** (config, effet immédiat).
Les `DynamicFunction` ont compilé et abouti **au 1er appel** — la vérification systématique des
signatures en source avant codage a évité toute erreur de compilation.

## 9. Récapitulatif des IDs

| Élément | Id |
|---------|-----|
| Fn `McpCreateEntityWorkflow` | `3a6030c9-7639-4558-9e6b-0c546f978422` |
| Fn `McpCreateIndicator` | `2e036c63-0032-47be-a987-9a2dc3526630` |
| Fn `McpCreateDashboard` | `b173685e-20ce-4353-bd90-0dd5aa31d85d` |
| Fn `McpAddIndicatorWidget` (corrigée) | `a6d9cb5e-4a0e-4a0b-a58a-3a42bb215890` |
| Fn `McpCreateMultiMeasureIndicator` | `383a2696-0b96-4b76-99ba-511a6ba4631c` |
| Fn `McpUpdateFunctionCode` | `2f28143c-f5e3-40cf-aff7-a727e68cb608` |
| Fn `McpCreateTypedIndicator` | `11c54e35-846d-4984-ada8-9c20cc41ea9f` |
| Fn `McpDiagReportData` (diagnostic rendu) | `39722aab-9148-48da-a674-47d9aa024332` |
| Fn `McpSetIndicatorChartType` | `a62366de-8c02-4b78-b3ce-b9fea3650eb5` |
| Dashboard « tous types » `MCP_DASH_ALLTYPES` | `9a0039c9-7ef1-4800-9b24-3e866a7416b7` |
| └ DynamicDashboard / DashCollection | `e989498b-…` / `f975c2d6-…` |
| └ 20 indicateurs (1 par ChartRepresentation) | cf. tableau §7 |
| EntityWorkflow (McpDemoVehicule) | `dd4f3cd7-080b-4d29-9dc4-3fa6996ba016` |
| Indicator (SUM Cout) | `c95f655a-ad63-4150-b311-3d5ba20964e2` |
| Indicator multi-mesures (SUM/AVG/COUNT) | `7fdfdec6-1d15-4ce4-bba1-4df5530008c3` |
| DynamicDashboard | `2ba1ed52-aaee-4b1f-9fdd-40ee49f41d1a` |
| DashCollection | `080358e4-a714-4e9d-96b7-3f83202d04bc` |
| Dashboard | `bdfa3383-816b-4f5d-9cd4-1d49f5fa46d7` |
| DashboardReportWidget (mono-mesure, ancien) | `97adb7b9-44ac-43dd-8683-b45e009384c0` |
| DashboardReportWidget (multi-mesures, nom OK) | `c681cb1e-ed77-4a24-a128-b45e00a25abf` |

---

## 10. ⚠️ `Grid` (0) = liste à plat ≠ `GridMatrice` (16) = tableau agrégé

**Constat** : le widget « Grid » (indicateur `0cbe03f4`) affichait **5 lignes brutes** d'une seule
colonne `Coût (€)` (149,90 / 590 / 100 / 320,50 / 100) — **aucun regroupement par voiture**, alors que
la dimension `IndicatorFilter` (`McpDemoVehicule.Immatriculation`, `FilterType=Aggregate`) était bien présente.

**Cause (source)** : `VPSoft.Services/Indicators/DataGenerator/Table/TableOutputGenerationStrategy.cs:21`
`bool isFlatList = options.Indicator.DefaultChartRepresentation == ChartRepresentation.Grid;`. Quand
`isFlatList == true`, la stratégie **court-circuite** `GetHorizontalAggregatedData` /
`GetVerticalAggregatedData` et appelle `GetNoAggregatedData` (`DataAggregator.cs:180`) → lignes brutes.
De même `TableHeaderGenerator.cs:23-112` ne génère **que** les colonnes de mesure (pas les dimensions)
quand `isFlatList`. `IndicatorDataGeneratorOptions.cs:30-66` ne classe en H/V que les filtres
`Aggregate|Both`. **Conclusion : un `Grid` pur n'agrège jamais.** Pour un tableau « coût par voiture »
(1 ligne par `Immatriculation` avec `SUM(Cout)`), il faut **`GridMatrice` (16)**.

**Correctif appliqué** : `McpSetIndicatorChartType("0cbe03f4-…", "GridMatrice")`. Vérifié via
`McpDiagReportData` → `chartRepresentation:16`, 2 colonnes (`Immatriculation`, `Coût (€)`) et **2 lignes
groupées** : `EX-100-PR → 320,50 €`, `EX-200-PR → 939,90 €` (= 149,90+590+100+100). Le widget
`c0226651` (dashboard `9a0039c9`, position 0/0) référence ce même `Indicator` → rendu mis à jour
immédiatement (aucun rebuild).

### Fonctions utilitaires créées pour le diagnostic
- **`McpDiagReportData`** (`39722aab-9148-48da-a674-47d9aa024332`) — `string indicatorId` →
  `IndicatorService.GetReportData(new IndicatorPreviewFormModel { IndicatorId = …, IsPreview = false })`
  sérialisé (JSON) : permet de **voir le rendu réel** (headers + data) d'un indicateur sans navigateur.
  ⚠️ `using` correct = `VPSoft.Domain.Contracts.Indicators` (un `using` inexistant ⇒ échec de compilation
  à chaud ⇒ retour `null` silencieux par le wrapper).
- **`McpSetIndicatorChartType`** (`a62366de-8c02-4b78-b3ce-b9fea3650eb5`) — `string indicatorId, string chartType` →
  `IndicatorService.GetSingle` + `Edit` (le PATCH générique sur `Indicator` renvoie **500**, comme pour `DynamicFunction`).

---

## 8. Workflow — TRANSITIONS (actions) : créer, exécuter, supprimer ✅ (2026-06)

### Modèle complet (verbatim `VPSoft.Domain/Models/Workflow/EntityWorkflowAction.cs:1-167`)
`EntityWorkflowAction { Code (unique), LocalizedName/LocalizedDescription (CultureParameters),
ValidationExpression(+Compiled), InjectionExpression(+Compiled) — cf. McpSetWorkflowActionCode §SKILL,
UserProperties (propriétés User de l'entité séparées par ';' : l'utilisateur courant doit y figurer),
**EntityWorkflowStatus (statut SOURCE)**, **NextEntityWorkflowStatus (statut CIBLE)**, Color (hex),
Order, IsEnabledComment (popup commentaire), Users : ISet<User>, Roles : ISet<RoleInModule> }`.
`EntityWorkflowStatus { LocalizedName, Code, Order, EntityName, EntityWorkflow }` — statuts
supplémentaires : `EntityWorkflowStatusService.Create` (même pattern + CultureParameter).

### Exécution (verbatim `EntityWorkflowService.cs:599-656`)
`ExecuteEntityWorkflowAction(IEntity entity, Guid buttonId, bool isFromFormModel = false, string? comment)` :
pose `entity.EntityWorkflowStatus = NextEntityWorkflowStatus` → **`CreateEntityWorkflowHistory`**
(comment inclus — c'est CETTE création d'historique qui déclenche les DataAlert de type Workflow) →
exécute l'`InjectionExpression` compilée (`InjectCustomEditOfWorkflow(entity, actionId)`).
**La persistance de l'entité est à la charge de l'appelant.** Visibilité des boutons côté UI :
statut courant = statut source ET (user ∈ Users OU rôle ∈ Roles OU user dans `UserProperties`)
(`GetEntityWorkflowActionButtons` `:229-328`) + business rules `Hide` sur Event=`EntityWorkflowAction(9)`
et `Activation` sur Event=`EntityWorkflow(8)` (`:485-569`).

### Fonctions MCP (code complet)

**`McpCreateWorkflowAction`** — `string entityName, string actionCode, string actionName,
string fromStatusCode, string toStatusCode, string roleCodesCsv, string color, string order` ·
CodeUsing : `using VPSoft.Domain.Models.Workflow;`
```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rm = AppDependencyResolver.GetService<IRepositoryManager>();
    if (sm.EntityWorkflowActionService.GetMany(x => x.Code == actionCode).Any()) return "ERR Code action deja existant: " + actionCode;
    var fromStatus = sm.EntityWorkflowStatusService.GetMany(x => x.Code == fromStatusCode && x.EntityName == entityName).FirstOrDefault();
    if (fromStatus == null) return "ERR statut source introuvable: " + fromStatusCode;
    var toStatus = sm.EntityWorkflowStatusService.GetMany(x => x.Code == toStatusCode && x.EntityName == entityName).FirstOrDefault();
    if (toStatus == null) return "ERR statut cible introuvable: " + toStatusCode;
    var action = new EntityWorkflowAction();
    action.Code = actionCode; action.EntityWorkflowStatus = fromStatus; action.NextEntityWorkflowStatus = toStatus;
    action.Color = string.IsNullOrEmpty(color) ? "#1f77b4" : color;
    action.Order = string.IsNullOrEmpty(order) ? 1 : int.Parse(order);
    action.IsEnabledComment = false;
    if (!string.IsNullOrEmpty(roleCodesCsv)) {
        var codes = roleCodesCsv.Split(',').Select(x => x.Trim()).Where(x => x != "").ToList();
        var roles = rm.RoleInModuleRepository.GetMany(x => codes.Contains(x.Code)).ToList();
        action.Roles = new HashSet<RoleInModule>(roles);
    }
    bool ok = sm.EntityWorkflowActionService.Create(action);
    var rc = new ResourceCultureJson { resourceKey = "LocalizedName", resourceValues = new List<ResourceCultureValueJson> {
        new ResourceCultureValueJson { cultureCode = "fr-FR", resourceValue = actionName },
        new ResourceCultureValueJson { cultureCode = "en-US", resourceValue = actionName } } };
    sm.CultureParameterService.SaveOrUpdate(rc, action.Id, typeof(EntityWorkflowAction).GetProperty("LocalizedName"));
    return JsonConvert.SerializeObject(new { success = ok, actionId = action.Id, code = actionCode, from = fromStatusCode, to = toStatusCode, roles = action.Roles != null ? action.Roles.Count : 0 });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**`McpRunWorkflowAction`** — `string entityName, string recordCode, string actionCode, string comment` ·
CodeUsing : `using NHibernate; using NHibernate.Criterion;` — charge l'enregistrement par Code
(NHibernate direct), exécute la transition, **persiste** (`session.Update` + `Flush`), renvoie le
nouveau statut :
```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var action = sm.EntityWorkflowActionService.GetMany(x => x.Code == actionCode).FirstOrDefault();
    if (action == null) return "ERR action introuvable (Code): " + actionCode;
    var session = NHSessionHelper.GetCurrentSession();
    var crit = session.CreateCriteria(entityName); crit.Add(Restrictions.Eq("Code", recordCode));
    var entity = crit.UniqueResult() as IEntity;
    if (entity == null) return "ERR enregistrement introuvable: " + entityName + " Code=" + recordCode;
    bool ok = sm.EntityWorkflowService.ExecuteEntityWorkflowAction(entity, action.Id, false, string.IsNullOrEmpty(comment) ? null : comment);
    session.Update(entity); session.Flush();
    dynamic d = entity;
    string newStatus = d.EntityWorkflowStatus != null ? (string)d.EntityWorkflowStatus.Code : "null";
    return JsonConvert.SerializeObject(new { success = ok, record = recordCode, action = actionCode, newStatus = newStatus });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

**`McpDeleteWorkflowAction`** — `string actionCode` → `EntityWorkflowActionService.Delete(action.Id)`.

### ✅ Roundtrip validé (démo, workflow « Cycle de vie » de McpDemoVehicule, EntityState=Inactive)
1. `McpCreateWorkflowAction("McpDemoVehicule","ZZMcp-Approve","Approuver (test MCP)",
   "McpDemoVehicule-Cycle de vie-A soumettre","McpDemoVehicule-Cycle de vie-Approuvé","<roleCode>","#28a745","1")` ✓
2. `create_entity` `ZZ-WF-1` → `McpRunWorkflowAction(...,"ZZMcp-Approve","Transition exécutée par le test MCP")`
   → `newStatus = "McpDemoVehicule-Cycle de vie-Approuvé"` ✓
3. `EntityWorkflowHistory` : 1 ligne avec le commentaire et le statut cible ✓ (les alertes Workflow
   s'accrochent à cette création d'historique).
4. `McpDeleteWorkflowAction` + suppression de l'enregistrement ✓ — un workflow **Inactive** reste
   pleinement exécutable par programme (seul l'AFFICHAGE est gaté par EntityState).

> Statuts : codes de la forme `"<Entity>-<Workflow>-<Statut>"` (ex. `McpDemoVehicule-Cycle de vie-Approuvé`) —
> les lister : `get_many_select("EntityWorkflowStatus","Code,LocalizedName,Order","EntityName == \"X\"")`.
> ⚠️ `EntityWorkflowAction` n'a pas de propriété `Name` (c'est `LocalizedName`).
