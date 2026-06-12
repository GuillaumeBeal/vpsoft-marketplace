# VPSoft — extraits de code source (digest synthétisé)

> **But** : rendre la skill autonome **en compréhension**. Les autres docs citent le code VPSoft en
> `fichier.cs:ligne` ; ce fichier embarque la **substance vérifiée** de ces références (enums + logique
> décisive), pour comprendre le *pourquoi* sans le dépôt. Ce sont des **extraits synthétisés** (valeurs
> et lignes vérifiées sur VPSoft 10.5 démo) — les `fichier:ligne` permettent de retrouver l'original
> exact, qui peut légèrement varier selon la version.

---

## 1. Enums décisifs

**`FormPropertyType`** (`VPSoft.Domain/Enums/FormPropertyType.cs`) — type d'un champ :
`String=0, Integer=1, DateTime=2, Boolean=3, Decimal=4, DynamicEnum=5, Enum=6, Entity=7, MultiCulture=8,
AutoIncrement=9, SectionName=10, DynamicMultiEnum=11, Date=12, File=13, Guid=14, EntityComment=15`.

**`FormPropertyCategory`** (même fichier) : `Standard=0, Formula=1, ExpressionField=3`.
- `Standard` = colonne normale. `Formula` = colonne SQL calculée. `ExpressionField` = propriété calculée **C#**.

**`RenderType`** (`VPSoft.Domain/Enums/RenderType.cs`) — rendu d'un champ :
`Input=0, Textarea=1, Editor=2, List=3, ListMultiple=4, CheckboxMultiple=5, RadioButton=6, Color=7,
File=8, Date=9, DateTime=10, MultiCulture=12, MultiCultureTextarea=13, MultiCultureEditor=14,
DynamicPage=16, FileImage=17, FileDocument=18, FileExcel=19, Password=20, Table=21, Tree=22, InlineEditor=23`.

**`FieldDisplay`** (`FieldDisplay.cs`) : `NewLine=0, InSuccession=1`.
**`DateDisplay`** (`DateDisplay.cs`) : `Date=0, DateTime=1`.

**`DynamicFieldRoleActionForList`** : `Inactive=0, Display=1, Displayable=2` (champ `Table`).
**`DynamicFieldRoleActionForAdd`** / **`DynamicFieldRoleActionForEdit`** (`DynamicFieldRole.cs:141-181`) :
`Inactive=0, Active=1, ActiveAndHidden=2, ReadOnly=3`.

**`IEDynamicFieldRole`** (`VPSoft.Domain/Models/ImportExport/IETableUsable.cs:510`) — import/export d'un champ :
`NotAvailable=0, Both=1, OnlyExport=2`.

**`MenuType`** (`VPSoft.Domain/Models/Builder/NavMenu.cs:176`) :
`SystemSettings=0, DynamicPage=1, EntityDynamic=2, …` (+ `DynamicDashboard`, `View`, `EntityDynamicCustom`).

**`EntityWorkflowStatusCategory`** : `ToBeSubmitted=0, Approved=1, Declined=2` (3 statuts par défaut).

**`UnitType`** (catégorie d'unité) : `Custom=0, Currency=1, Surface=2, Length=3, Volume=4, Mass=5,
Pressure=6, Temperature=7, Energy=8, Speed=9, Time=10, …`.

**`ChartRepresentation`** (`VPSoft.Domain/Models/Indicators/Indicator.cs:149`) :
`Grid=0, Bar=1, HorizontalBar=2, Lines=3, Area=4, Pie=5, Radar=6, StackedBar=7, …, GridMatrice=16,
Card=17, StackedBar100=18, Doughnut=20, HalfDoughnut=21`. ⚠️ `Grid` = liste à plat (n'agrège pas) ;
`GridMatrice` = pivot agrégé.

---

## 2. Champ expression C# — génération (`ClassBuilder.cs:153-180`)
Pour `Category=ExpressionField`, le générateur émet une **méthode C#** dont le corps EST le `ReferenceLambda` :
```csharp
private int _propName;
private int PropNameFunction(int value) {
    <ReferenceLambda>   // l.168 : DOIT contenir "return", sinon un throw est généré
}
public virtual int PropName { get => _propName; set { _propName = PropNameFunction(value); /*...*/ } }
```
→ `ReferenceLambda` = **corps de méthode** (`"return Kilometrage + KilometrageAnnuel;"`), pas `x => …`.
Recalcul via `RefreshExpressionFields()` (`ClassBuilder.cs:372-385`, réaffecte `Prop = Prop;`).

## 3. Mapping par catégorie (`ClassMappingBuilder.cs:175-193, 293-296`)
```csharp
case FormPropertyCategory.Formula:        // l.177-178 : colonne SQL calculée
    Map(x => x.Prop).Formula(@"(<Formula>)");
case FormPropertyCategory.ExpressionField: // l.181-190 : .Access.CamelCaseField (backing field C#)
    Map(x => x.Prop, "<col>").Access.CamelCaseField(Prefix.Underscore);
// l.293-296 : si un champ porte une Unit → relation mesures
if (properties.Any(x => x.Unit != null))
    HasMany(x => x.DynamicMeasures).KeyColumn("EntityId")...;
```
⚠️ Formule SQL = noms de **colonnes physiques** `FormBuilder_<Prop>` (pas le PropertyName).

## 4. Unité — règle de rebuild (`DynamicFieldService.cs:3128-3161`)
`formProperty.Unit = UnitRepository.LoadReference(unitId)`. Le code ne marque MODIFIED (→ besoin schéma
`DynamicMeasures`) **que** la 1ʳᵉ unité de l'entité (`unitAdded && !entityIsWithUnit && Status==OK`).
**MAIS** le formatage avec l'unité est généré au build + `DynamicField` est `[EntityCache]` → en pratique
**`McpBuild` requis à chaque pose/changement d'unité** pour l'affichage.

## 5. Visibilité champ — consommateur autoritaire
`DynamicFieldService.GetDynamicFieldsByAction(FieldAction, …)` (`DynamicFieldService.cs:1771`) :
- `Table` (liste) : `x.Table != Inactive` · `Create` : `RoleActionForAdd != Inactive` ·
  `Edit` : `RoleActionForEdit != Inactive` · `Detail` : `x.CanRead`.
Écriture du mapping rôle↔champ : `DynamicSettingsService.SaveDynamicSettingsRolesPermissions`
(`DynamicSettingsService.cs:3420`) : `dfr.Table = RoleActionForTable`, `dfr.Api = …`, `dfr.ImportExport = …`.
Exposition API : `DynamicApiService.GetUsableTable<T>()` filtre `.Where(x => x.Api)` (`:306`).

## 6. Sections (formulaire) — `DynamicSectionService.cs:438-476`
`MoveFieldsToFromSections` ajoute/retire des `DynamicSectionPropertyName { PropertyName, OrderBy, DynamicSection }`
dans `DynamicSection.DynamicSectionPropertyNames`, puis `Edit(section)`. Un champ **sans** section
n'apparaît pas au formulaire (même si visible en liste).

## 7. Reverse-link (`FormBuilderController.cs:1428-1551`)
Forward : `FormProperty { Type=Entity, EntityNameSelected, EntityFullName, ReferenceRelation,
EntityTableName, InverseReferencePropertyName, Nullable=true }`. Puis création de la propriété **inverse**
sur l'entité cible (`InverseReference=true`, relation inversée `HasMany↔Reference`, câblage croisé
`EntityFullName/EntityTableName/TableName`). `ReferenceRelation` : `Reference=0, HasMany=1, HasManyToMany=2, HasOne=3`.

## 8. API REST dynamique (`DynamicApiService.cs`)
`Post<T>` (`:152`) = import **Création** (JSON plat) ; `Patch<T>` (`:195`) = import **Modification**.
- **Référence en create** : passer le champ **= code de la cible** (`{"Vehicule":"DEMO-EXPR2"}`), pas `_Code_`.
- **`update_entity_patch` = 500** sur entité à reverse-link : **référence circulaire** chargée par l'import
  Modification → **Mapster** boucle (pas de `PreserveReference`). Côté Json.NET c'est gardé
  (`JsonSerializerHelper.cs:471-472,592-593` : `PreserveReferencesHandling`), côté Mapster non.
  **Contournement** : poser la référence en NHibernate direct (`McpSetReference`,
  `NHSessionHelper.GetCurrentSession()` + `session.CreateCriteria(...).Add(Restrictions.Eq("Code", …))`).
  **Fix appli recommandé** : `TypeAdapterConfig.GlobalSettings.Default.PreserveReference(true)` / `MaxDepth`.

## 9. Menu (`NavMenuService.cs`, `NavMenu.cs`, `DynamicModuleService.cs`)
- `NavMenu` (sous-classe `NavMenuDynamicSettings` pour une table) rattaché à `DynamicModule`,
  `IsActiveUser`/`IsActiveSetting`, `OrderBy`, **`RolePermissions` (ISet<RoleInModule>) = profils qui voient l'item**.
- `ActivateOrCreate(NavMenuCreateOrActivateFormModel{Name,EntityId,DynamicModuleId,Type=EntityDynamic,IsForSettings,Order})`
  (`:478`) crée le NavMenu (`IsActiveUser=true`) **mais ne pose PAS `RolePermissions`** → le faire ensuite.
- Arbre par utilisateur : `DynamicModuleService.GetNavMenusItemsFromModuleId` filtre
  `NavMenu.RolePermissions ∩ user.RoleInModules` (+ entité `RolePermission.CanSee`).
- ⚠️ `McpCreateTable` ne crée PAS de NavMenu (contrairement à l'UI `AddDynamicEntity`) → `McpAddTableToMenu`.

## 10. Permissions d'entité (`RolePermission`)
`RoleInModule` (rôle↔module) → `RolePermission { EntityName, ViewName, CanSee/CanAdd/CanEdit/CanDelete }`.
Sans `RolePermission` (CanSee/…), l'API REST renvoie **HTTP 500**. Posé par `McpGrantTablePermission`.

## 11. Workflow (`EntityWorkflow*`, `EntityWorkflowService.cs`)
- `EntityWorkflow { Name, EntityName, EntityState }` + `EntityWorkflowStatus` + `EntityWorkflowAction`
  (`ValidationExpression`/`InjectionExpression` C#, `Roles`/`Users`).
- **Affichage gaté** par : enregistrement avec `EntityWorkflowStatus != null` **ET**
  `EntityWorkflow.EntityState == Active` (`GetEntityWorkflowIdFromEntity` filtre `EntityState==Active`).
- **Création de table n'active aucun workflow.** Créer un workflow = `Create` + 3×`CreateEntityWorkflowDefautStatus`.
- ⚠️ **Quasi impossible à supprimer** (cascade statuts/actions/historique). **Masquer** = `EntityState=Inactive`
  via `EntityWorkflowService.Edit` (réversible) → `McpSetWorkflowActive`.

## 12. Libellés / descriptions / unités (données de référence)
- Libellé/infobulle : **`DynamicFieldName { Name, ColumnName, Tooltip, Culture }`** (1 ligne/culture) —
  `DynamicFieldNameService`. Description : **`DynamicFieldDescription { Description, Culture }`** —
  `DynamicFieldDescriptionService`. Le `DynamicField` lui-même n'a PAS de Label.
- Unité = donnée de réf : **`Unit { Name, Code(unique), UnitType }`** + **`Measure { Symbol, Code, Value,
  IsBasicUnit, Unit, UnitType }`**. `UnitService.CreateUnit(UnitCreateFormModel)` (`UnitService.cs:297`)
  **ne crée PAS de mesure** (sauf type Derived) → créer la `Measure` base (`Symbol="km", Value=1,
  IsBasicUnit=true`) à part via `MeasureService.Create`. Écran UI : `/System/Settings/Unit`.

## 13. DynamicFunction (le moteur)
`DynamicFunction { Name, Code, Parameters, ReturnValue, CodeUsing, CodeFunction, CodeClass, IsAsync, DynamicFolder }`,
compilée à chaud (`CSharpEngineService`) au 1er appel. Le wrapper **injecte** `IServiceManager`/`AppDependencyResolver`
et **avale les exceptions** (retour `default`→`null`) → d'où l'obligation du **try/catch interne** qui renvoie
`ex.Message`. Invocation via `POST /Api/V2/VP/Functions/Invoke` (outil MCP `invoke_vpsoft_function`).
