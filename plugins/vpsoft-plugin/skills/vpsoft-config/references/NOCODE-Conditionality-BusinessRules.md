# VPSoft — Conditionnalités de champs & Business Rules NO-CODE (via MCP)

> **But** : créer/lire/supprimer des **business rules** (masquer/afficher/lecture seule/obligatoire/
> valider/filtrer…) et des **conditionnalités de champs** SANS UI, par programme via MCP — de façon
> **autoportante** (modèle complet, JSON des arbres, opérateurs, fonctions `Mcp*` prêtes).
> Vérifié sur le source 10.5 (`fichier:ligne`) et **testé sur la démo** quand marqué ✅.
> (Moteur interne détaillé — folding NH, sandbox Dynamic LINQ — : `docs/BusinessRuleModule.md` du dépôt.)

**Sommaire** : §1 modèle mental · §2 entités & enums (sous-classes, Event/Type, cibles par Event) ·
§3 stockage des arbres & RuleSet · §4 JSON `RuleNodeDto` + exemples réels · §5 langage (roots,
opérateurs, fonctions) · §6 évaluation client/serveur & services · §7 fonctions MCP
(`McpCreateBusinessRule` tous kinds, `McpSetBusinessRuleExtras`, `McpGetRuleTree`, `McpDeleteRuleTree`,
`McpDeleteBusinessRule` + tests ✅) · §8 recettes · §9 lecture/audit · §10 gotchas.

---

## 1. Modèle mental

Une **business rule** = `BusinessRule` (QUOI faire, SUR QUOI, QUAND) + 1..n **arbres de règles**
(`RuleTree` : LA CONDITION, stockée normalisée, échangée en JSON `RuleNodeDto`).

```
BusinessRule (HideRule, ValidationRule, …)
 ├─ EntityId  ───────────► la CIBLE (DynamicField.Id, DynamicSection.Id, DynamicSettings.Id, DynamicButton.Id…)
 ├─ Event     ───────────► le TYPE de cible (Field, Section, Entity, DynamicButton, NavMenu, …)
 ├─ Type      ───────────► l'ACTION (Hide, Show, ReadOnly, Mandatory, Validation, Filter, …) — posé par le ctor
 ├─ ViewName  ───────────► null = la table maître ; sinon la vue précise
 ├─ Priority  ───────────► ordre d'évaluation
 └─ ActivationRuleTreeId ► l'arbre CONDITION (+ ActionRuleTreeId / AffectationRuleTreeId selon le type)
```

**Une « conditionnalité de champ » = une BusinessRule `Hide/Show/ReadOnly/Mandatory` avec
`Event=Field` et `EntityId = DynamicField.Id`.** ✅ Vérifié démo : la règle
« Hide equipment while Orga isn't selected » est une `HideRule { EntityName="Ticket", Event=Field,
EntityId = <Id du DynamicField Ticket.Equipment> }`.

> **Legacy** : les entités `Conditionality`/`ConditionalityAction`
> (`VPSoft.Domain/Models/BusinessRule/Legacy/`) sont l'ancien moteur, conservées pour compat.
> **Tout nouveau besoin passe par BusinessRule + RuleTree.**

---

## 2. Entités & enums (verbatim `VPSoft.Domain/Models/BusinessRule/RuleTree/BusinessRule.cs`)

### 2.1 Classe abstraite

```csharp
public abstract class BusinessRule : EntityAudit, IAglxEntity, ITemplatedEntity<BusinessRule>
{
    public virtual string? Name { get; set; }
    public virtual string? Description { get; set; }
    public virtual Guid EntityId { get; set; }            // la cible (selon Event)
    public virtual string? EntityName { get; set; }       // entité métier concernée
    public override string? ViewName { get; set; }        // null = master ; sinon vue
    public virtual BusinessRuleEvent Event { get; set; }
    public virtual BusinessRuleType Type { get; set; }    // posé par le constructeur de la sous-classe
    public virtual int Priority { get; set; }
    public virtual BusinessRule? TemplateEntity { get; set; }  // héritage template/override
}
public interface IActivableRule { Guid? ActivationRuleTreeId { get; set; } }   // :298-301
```

### 2.2 Sous-classes scellées (1 table TPH, discriminant = Type)

| Classe | Ctor pose `Type` | Propriétés propres |
|---|---|---|
| `HideRule` (:71) / `ShowRule` | `Hide=2` / `Show=3` | `ActivationRuleTreeId` |
| `ReadOnlyRule` / `MandatoryRule` | `ReadOnly=5` / `Mandatory=6` | `ActivationRuleTreeId` (+ `MandatoryMessageKey`) |
| `ValidationRule` | `Validation=1` | `ActivationRuleTreeId` + `ValidationMessageKey` (clé message d'erreur) |
| `FilterRule` | `Filter=0` | `ActivationRuleTreeId` + **`ActionRuleTreeId`** (l'arbre de filtre lui-même) |
| `AffectationRule` | `Affectation=4` | `ActivationRuleTreeId` + **`AffectationRuleTreeId`** — ⚠️ malgré son nom, cet id pointe un **`ParameterValue` (spec)**, PAS un RuleTree : la « valeur à affecter » est un `ParameterValueDto` persisté par `BusinessRuleService.PersistAffectationSpec(existingId, spec)` (`BusinessRuleService.cs:1149-1161` → `RuleTreeRepository.SaveParameterValue`) et supprimé par `DeleteAffectationSpec` (✅ vérifié en réel) |
| `PropagationRule` | `Propagation=9` | `DynamicFieldId` (champ imbriqué cible) + `PropagationProperties : IList<PropagationProperty>` |
| `AlertRule` | `Alert=8` | Activation + Affectation + Action trees |
| `ActivationRule` | `Activation=7` | `ActivationRuleTreeId` |

### 2.3 Enums (valeurs numériques — utilisables dans les `where` MCP)

**`BusinessRuleEvent`** (`:125-173`) — sur quoi porte la règle :
`Entity=0, Field=1, Section=2, DynamicButton=3, Alert=4, NavMenu=5, Dashboard=6, Mail=7,
EntityWorkflow=8, EntityWorkflowAction=9, Scope=10`.

**`BusinessRuleType`** (`:179-220`) — l'action :
`Filter=0, Validation=1, Hide=2, Show=3, Affectation=4, ReadOnly=5, Mandatory=6, Activation=7,
Alert=8, Propagation=9`.

### 2.4 Cible (`EntityId`) selon `Event` — table de résolution

| Event | `EntityId` = | Résolution MCP (lecture) |
|---|---|---|
| `Entity=0` | `DynamicSettings.Id` | `get_many_select("DynamicSettings","Id","EntityName == \"X\" and ViewName == null")` |
| `Field=1` | **`DynamicField.Id`** | `get_many_select("DynamicField","Id","EntityName == \"X\" and PropertyName == \"Y\"")` |
| `Section=2` | `DynamicSection.Id` | `get_many_select("DynamicSection","Id,Name","DynamicSettings.EntityName == \"X\"")` |
| `DynamicButton=3` | `DynamicButton.Id` | `get_many_select("DynamicButton","Id","Code == \"…\"")` |
| `NavMenu=5` / `Dashboard=6` / `EntityWorkflow[Action]=8/9` | l'Id de l'objet correspondant | idem par entité |

---

## 3. Stockage des arbres (normalisé) & RuleSet

Un arbre est stocké en lignes (`VPSoft.Domain/Models/BusinessRule/RuleTree/`) — **PAS un blob JSON** :

| Table | Rôle | Colonnes clés |
|---|---|---|
| `RuleNode` | nœud | `RuleTreeId` (= l'Id porté par `ActivationRuleTreeId` etc.), `ParentNodeId`, `NodeType` (`NodeKind` : Group=0, Rule=1, Expression=2, RuleSet=3), `SortOrder`, `RuleSetId?` |
| `RuleGroup` | données d'un Group | `NodeId`, `Combinator` (And=0, Or=1), `Not` |
| `RuleClause` | données d'une Rule | `NodeId`, `Field` (chemin, ex. `entity.Organization`), `Operator`, `Expression?` |
| `ParameterValue` | valeur (RHS / argument) | `RuleNodeId?`, `Ordinal`, `Source` (`ValueSource` : LITERAL=0, MODEL=1, CONTEXT=2, FUNCTION=3, PARAM=4, EXPRESSION=5, PROPERTY=6), `StringValue/NumberValue/BooleanValue/DateValue`, `FunctionId?` |
| `FunctionCall` / `FunctionArgument` | appel de fonction (`Lower`, `DateAdd`…) | `Name`, `ReturnType` / `FunctionId`, `ValueId`, `ParamName` |
| `RuleSet` | **arbre réutilisable nommé** | `Name`, `Description`, `EntityName` — son arbre est stocké avec `RuleTreeId = RuleSet.Id` ; référencé dans d'autres arbres par un nœud `kind=ruleset` |

> ⚠️ **`RuleNode` & co ne sont PAS des `IEntity`** → **non requêtables via `get_many_select`**
> (erreur `violates the constraint of type parameter 'TEntity'` — ✅ constaté). Lire/écrire les arbres
> passe par **`IRuleTreeService`** (→ fonctions `Mcp*` §6). Les `BusinessRule`, elles, SONT requêtables.

---

## 4. Le JSON d'un arbre — `RuleNodeDto` (`VPSoft.Domain/Contracts/Rule/Core/RuleNodeDto.cs:1-169`)

```jsonc
{
  "id": "guid-N",            // facultatif à l'écriture (régénéré par SaveOrReplace)
  "kind": 0,                 // NodeKind : 0=group, 1=rule, 2=expression, 3=ruleset
  "combinator": 0,           // groupes : 0=and, 1=or
  "not": false,              // groupes : inverse le résultat
  "children": [ … ],         // groupes : sous-nœuds
  "field": "entity.Champ",   // rules : chemin du membre gauche
  "operator": "gt",          // rules : opérateur (cf. §5)
  "values": [                // rules : opérandes droits (0..n selon l'arité)
    { "source": 0, "value": 100000 }      // ValueSource.LITERAL
  ],
  "function": { "name": "Lower", "returnType": "...", "parameterValues": { … } },  // optionnel (transforme le membre gauche)
  "expression": "row.Name != null && row.Name.StartsWith(\"x\")",                  // kind=2 : Dynamic LINQ
  "ruleSetId": null, "ruleSetName": null                                           // kind=3 : référence RuleSet
}
```

### ✅ Exemples RÉELS lus sur la démo (`RuleTreeService.Load`)

**Conditionnalité « cacher le champ Equipment tant que l'Orga n'est pas choisie »**
(HideRule sur Ticket.Equipment, Event=Field) :
```json
{ "kind": 0, "combinator": 0, "not": false, "children": [
    { "kind": 1, "field": "entity.Organization", "operator": "isEmpty", "values": [] } ] }
```

**« hide si GaranteeBank == false »** (HideRule sur LegalLease, littéral booléen sous forme string) :
```json
{ "kind": 0, "combinator": 0, "children": [
    { "kind": 1, "field": "entity.GaranteeBank", "operator": "equals",
      "values": [ { "source": 0, "value": "false" } ] } ] }
```

**✅ Créé + relu + supprimé par MCP (test roundtrip complet)** — « cacher KilometrageAnnuel si
Kilometrage > 100000 » sur `McpDemoVehicule` :
```json
{ "kind": 0, "combinator": 0, "not": false, "children": [
    { "kind": 1, "field": "entity.Kilometrage", "operator": "gt",
      "values": [ { "source": 0, "value": 100000 } ] } ] }
```
(relu : `value` ressort en décimal `100000.0000000000` — stocké dans `ParameterValue.NumberValue`.)

---

## 5. Langage des règles (roots, opérateurs, fonctions) — condensé opérationnel

### 5.1 Roots (côté gauche `field` ET côté valeurs `source=MODEL/PROPERTY`)

`entity.X` (la ligne courante ; un chemin **non préfixé** = `entity.`) · `oldEntity.X` (avant édition) ·
`parentEntity.X` (modèle parent, ex. tableau imbriqué) · `global.X` (contexte : `global.Time`,
utilisateur…). **Casse STRICTE des roots** (`entity`, pas `Entity`) ; les noms de propriétés, eux,
sont insensibles à la casse.

### 5.2 Opérateurs (avec arité)

| Famille | Opérateurs |
|---|---|
| Comparaison | `equals`, `notEquals`, `gt`, `ge`, `lt`, `le`, `between` (2 valeurs, bornes incluses) |
| Chaînes | `contains`, `notContains`, `startsWith`, `endsWith`, `equalsIgnoreCase`, `isEmpty`, `isNotEmpty` (0 valeur) |
| Dates (synonymes) | `before`/`after`/`onOrBefore`/`onOrAfter` (= `<`,`>`,`<=`,`>=`) |
| Ensembles | `in`, `notIn` (variadique ≥1) |
| Collections/enums | `hasFlag`, `notHasFlag`, `containsAny`, `containsAll`, `notContainsAny` |
| Null/bool | `isNull`, `isNotNull` (0 val.), `isTrue`, `isFalse` (0 val.) |
| Arbres (closure table) | `tree.isChildOf`, `tree.isParentOf` (RHS : chemin d'entité arbre, `Code`/`Id` scalaire, ou lambda `(org => org.Code.StartsWith("EMEA-"))`) |
| Legacy | `none` (true si un segment du chemin est null) |

Arité stricte : unaire = 0 valeur · binaire = 1 · ternaire = 2 · variadique ≥ 1.

### 5.3 Fonctions (registre `RuleBootstrap.DefaultFunctionRegistry`)

Chaînes : `Lower, Upper, Length, Substring, Trim, Concat` · Dates : `Today, Now, DateAdd` ·
Math : `Abs, Round` · Util : `IsNull`. Trois formes : transformer le champ gauche
(`Length(Name) gt 3` → `function` sur le nœud), LHS pure fonction, ou valeur RHS
(`Name equals Lower("JOHN")` → `source=FUNCTION` + `FunctionCall`).
`Today`/`Now` lisent `global.Time` (horloge injectable → tests déterministes).

### 5.4 Nœuds `expression` (kind=2, Dynamic LINQ)

Paramètres disponibles : `(row, global, model, oldModel, parentModel, ps)`. Sandbox : pas d'appels
`object.*` sur `ps[...]` ; préférer `ps.ContainsKey("flag")` / comparaisons ; court-circuiter les nulls.
Groupe vide ⇒ `true`. `not` inverse le groupe.

---

## 6. Évaluation — qui applique quoi, où

| Type | Côté | Mécanisme |
|---|---|---|
| `Hide/Show/ReadOnly/Mandatory` (Event=Field/Section/Button) | **Client (formulaire)** | le front charge les règles de la cible + arbres et applique l'état au fil de la saisie (composants `VPSoft.Web/src/components/rule/**`, `rule-types/*.vue`, store `ruleBuilder.ts`) |
| `Validation` | **Serveur au save** (+ client) | message via `ValidationMessageKey` (clé ressource) |
| `Filter` (Event=Entity/Field) | **Serveur requêtes NH** | l'arbre est compilé en `Expression<Func<T,bool>>` (`RulePredicateBuilder` + `RulePredicateBinder.BindPredicateForNh` — folding des sous-arbres non-row en constantes) et appliqué aux listes/données ; sur un champ référence (Event=Field), il **filtre la liste déroulante** |
| `Affectation` | Serveur (save/workflow) | l'arbre d'affectation CALCULE la valeur posée sur le champ cible |
| `Propagation` | Serveur (save parent→enfants) | `PropagationProperties` = mappings propriété→`ParameterValue` |

**Services** (`IServiceManager`) : `BusinessRuleService` (`IBusinessRuleService` —
`Create/Edit/Delete/DeleteWithDependencies/GetSingle/GetMany/GetJoinWhereForBusinessRule(entityId, event, viewName)/
GetReactionsFromBusinessRules`) et **`RuleTreeService`** (`IRuleTreeService` —
`Load(ruleTreeId)`, `SaveOrReplace(ruleTreeId, RuleNodeDto)`, `Delete(ruleTreeId)`,
`SaveRuleSet/LoadRuleSet/GetAllRuleSets`). REST (UI no-code) : `POST/GET/DELETE /api/v1/BusinessRules[...]`
et `/api/v1/RuleTrees[...]` (`VPSoft.Presentation/Controllers/Api/BusinessRuleController.cs`,
`RuleTreeController.cs`).

**Effet immédiat : config pure, AUCUN rebuild** (F5 côté UI).

---

## 7. ✅ Fonctions MCP (code complet, validées sur la démo)

> Conventions habituelles : params **string**, try/catch interne, `AppDependencyResolver`.
> À (re)créer via `McpCreateFunction(name, parameters, "System.Object", codeUsing, codeFunction, "false", "", folderId)`.

### `McpGetRuleTree` — lire un arbre (JSON)
- **Parameters** : `string ruleTreeId`
- **CodeUsing** : `using VPSoft.Services.Abstractions; using VPSoft.Services.Abstractions.Rule; using Newtonsoft.Json;`
```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var tree = sm.RuleTreeService.Load(Guid.Parse(ruleTreeId));
    if (tree == null) return "null tree";
    return JsonConvert.SerializeObject(tree);
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

### `McpCreateBusinessRule` — créer règle + arbre d'activation (TOUS les kinds)
- **Parameters** : `string kind, string entityName, string eventName, string targetEntityId, string ruleName, string activationTreeJson, string viewName`
- **CodeUsing** : `using VPSoft.Domain.Models.BusinessRule.RuleTree; using VPSoft.Domain.Contracts.Rule.Core; using VPSoft.Services.Abstractions; using Newtonsoft.Json;`
- `kind` ∈ `Hide|Show|ReadOnly|Mandatory|Validation|Filter|Affectation|Activation|Alert|Propagation`
```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    BusinessRule rule = null;
    if (kind == "Hide") rule = new HideRule(); else if (kind == "Show") rule = new ShowRule();
    else if (kind == "ReadOnly") rule = new ReadOnlyRule(); else if (kind == "Mandatory") rule = new MandatoryRule();
    else if (kind == "Validation") rule = new ValidationRule(); else if (kind == "Filter") rule = new FilterRule();
    else if (kind == "Affectation") rule = new AffectationRule(); else if (kind == "Activation") rule = new ActivationRule();
    else if (kind == "Alert") rule = new AlertRule(); else if (kind == "Propagation") rule = new PropagationRule();
    else return "ERR kind inconnu: " + kind;
    rule.Name = ruleName; rule.EntityName = entityName;
    rule.EntityId = Guid.Parse(targetEntityId);
    rule.Event = (BusinessRuleEvent)Enum.Parse(typeof(BusinessRuleEvent), eventName);
    rule.ViewName = string.IsNullOrEmpty(viewName) ? null : viewName;
    rule.Priority = 0;
    Guid? treeId = null;
    if (!string.IsNullOrEmpty(activationTreeJson)) {
        var dto = JsonConvert.DeserializeObject<RuleNodeDto>(activationTreeJson);
        treeId = Guid.NewGuid();
        sm.RuleTreeService.SaveOrReplace(treeId.Value, dto);
        var pAct = rule.GetType().GetProperty("ActivationRuleTreeId");   // par réflexion : AffectationRule
        if (pAct != null) pAct.SetValue(rule, treeId);                    // n'implémente pas IActivableRule
    }
    bool ok = sm.BusinessRuleService.Create(rule);
    return JsonConvert.SerializeObject(new { success = ok, ruleId = rule.Id, ruleType = rule.GetType().Name, treeId = treeId });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

### `McpSetBusinessRuleExtras` — compléments par type (arbres secondaires, spec, messages, priorité)
- **Parameters** : `string ruleId, string extrasJson`
- **CodeUsing** : `using VPSoft.Domain.Models.BusinessRule.RuleTree; using VPSoft.Domain.Contracts.Rule.Core; using VPSoft.Services.Abstractions; using Newtonsoft.Json;`
- `extrasJson` (toutes clés optionnelles) :
  `{"activationTree":<RuleNodeDto>, "actionTree":<RuleNodeDto> /*FilterRule|AlertRule*/,
    "affectationSpec":<ParameterValueDto> /*AffectationRule : {"source":0|1|2|3,"value":…}*/,
    "validationMessageKey":"…", "mandatoryMessageKey":"…", "priority":N, "dynamicFieldId":"guid" /*PropagationRule*/}`
- Réutilise l'id existant si déjà posé (remplacement en place), sinon en génère un.
```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    var rule = sm.BusinessRuleService.GetSingle(Guid.Parse(ruleId));
    if (rule == null) return "ERR BusinessRule introuvable: " + ruleId;
    var jo = JObject.Parse(extrasJson); var done = new List<string>(); var t = rule.GetType();
    string[] treeProps = new string[] { "activationTree|ActivationRuleTreeId", "actionTree|ActionRuleTreeId" };
    foreach (var tp in treeProps) { var parts = tp.Split('|');
        if (jo[parts[0]] != null) {
            var p = t.GetProperty(parts[1]); if (p == null) return "ERR " + parts[1] + " non supporte par " + t.Name;
            var dto = jo[parts[0]].ToObject<RuleNodeDto>();
            var cur = (Guid?)p.GetValue(rule); var tid = cur ?? Guid.NewGuid();
            sm.RuleTreeService.SaveOrReplace(tid, dto); p.SetValue(rule, tid); done.Add(parts[1] + "=" + tid); } }
    if (jo["affectationSpec"] != null) {
        var p = t.GetProperty("AffectationRuleTreeId"); if (p == null) return "ERR AffectationRuleTreeId non supporte par " + t.Name;
        var spec = jo["affectationSpec"].ToObject<ParameterValueDto>();
        var cur = (Guid?)p.GetValue(rule);
        var sid = sm.BusinessRuleService.PersistAffectationSpec(cur, spec);    // ← spec ParameterValue, PAS un arbre
        p.SetValue(rule, sid); done.Add("affectationSpec=" + sid); }
    if (jo["validationMessageKey"] != null) { var p = t.GetProperty("ValidationMessageKey"); if (p == null) return "ERR ValidationMessageKey non supporte par " + t.Name; p.SetValue(rule, (string)jo["validationMessageKey"]); done.Add("validationMessageKey"); }
    if (jo["mandatoryMessageKey"] != null) { var p = t.GetProperty("MandatoryMessageKey"); if (p == null) return "ERR MandatoryMessageKey non supporte par " + t.Name; p.SetValue(rule, (string)jo["mandatoryMessageKey"]); done.Add("mandatoryMessageKey"); }
    if (jo["priority"] != null) { rule.Priority = (int)jo["priority"]; done.Add("priority=" + rule.Priority); }
    if (jo["dynamicFieldId"] != null) { var p = t.GetProperty("DynamicFieldId"); if (p == null) return "ERR DynamicFieldId non supporte par " + t.Name; p.SetValue(rule, Guid.Parse((string)jo["dynamicFieldId"])); done.Add("dynamicFieldId"); }
    bool ok = sm.BusinessRuleService.Edit(rule);
    return JsonConvert.SerializeObject(new { success = ok, ruleId = rule.Id, ruleType = t.Name, applied = done });
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

### `McpDeleteRuleTree` — supprimer un arbre isolé (orphelin, RuleSet…)
- **Parameters** : `string ruleTreeId` · **CodeUsing** : `using VPSoft.Services.Abstractions; using VPSoft.Services.Abstractions.Rule;`
```csharp
try { var sm = AppDependencyResolver.GetService<IServiceManager>();
    sm.RuleTreeService.Delete(Guid.Parse(ruleTreeId)); return "deleted tree " + ruleTreeId;
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message; }
```

### `McpDeleteBusinessRule` — supprimer règle + arbres (cascade propre)
- **Parameters** : `string ruleId` · **CodeUsing** : `using VPSoft.Services.Abstractions;`
```csharp
try {
    var sm = AppDependencyResolver.GetService<IServiceManager>();
    sm.BusinessRuleService.DeleteWithDependencies(Guid.Parse(ruleId));
    return "deleted " + ruleId;
} catch (Exception ex) { return "ERR " + ex.GetType().FullName + ": " + ex.Message + (ex.InnerException != null ? " | inner: " + ex.InnerException.Message : ""); }
```

### ✅ Tests roundtrip COMPLETS validés sur la démo (2026-06)

**Hide (conditionnalité de champ)** :
1. Cible : `get_many_select("DynamicField","Id","EntityName == \"McpDemoVehicule\" and PropertyName == \"KilometrageAnnuel\"")` → `8f40ab6e-…`.
2. `McpCreateBusinessRule("Hide","McpDemoVehicule","Field","8f40ab6e-…","ZZ Test…","{\"kind\":0,\"combinator\":0,\"not\":false,\"children\":[{\"kind\":1,\"field\":\"entity.Kilometrage\",\"operator\":\"gt\",\"values\":[{\"source\":0,\"value\":100000}]}]}","")`
   → `{success:true, ruleId:3bd0ca2d-…, treeId:a72d462e-…}`.
3. `McpGetRuleTree(treeId)` → arbre identique (ids normalisés — `SaveOrReplace` clone et régénère les ids).
4. `McpDeleteBusinessRule(ruleId)` → `deleted` ; re-lecture : `null tree` ; `get_entities_count("HideRule","EntityName == \"McpDemoVehicule\"")` → 0.
   **`DeleteWithDependencies` supprime bien la règle ET ses arbres.**

**Affectation (arbre d'activation + spec + priorité)** :
1. `McpCreateBusinessRule("Affectation","McpDemoVehicule","Entity","<dsId>","ZZ Test Affectation v2","{\"kind\":0,\"combinator\":0,\"not\":false,\"children\":[]}","")` → `ruleType:"AffectationRule"`.
2. `McpSetBusinessRuleExtras(ruleId, "{\"affectationSpec\":{\"source\":1,\"value\":\"entity.KilometrageAnnuel\"},\"priority\":7}")`
   → `applied:["affectationSpec=c29aeb04-…","priority=7"]` (source=1=MODEL : copie un champ).
3. Relecture : `get_many_select("AffectationRule","Name,Priority,ActivationRuleTreeId,AffectationRuleTreeId",…)` ✓.
4. `McpDeleteBusinessRule` → arbre d'activation supprimé (`null tree`) ET spec supprimée (`DeleteAffectationSpec`), count 0. ✅

---

## 8. Recettes no-code

### Conditionnalité de champ (« cacher X si … »)
```
0) select_app("demo-vesta-10-5")
1) targetId = get_many_select("DynamicField","Id","EntityName == \"<Table>\" and PropertyName == \"<Champ>\"")
2) McpCreateBusinessRule("Hide"|"Show"|"ReadOnly"|"Mandatory", "<Table>", "Field", targetId, "<nom parlant>",
   '<RuleNodeDto JSON>', "")          // ViewName "" = master ; sinon nom de la vue
3) Vérifier : get_many_select("HideRule","Name,EntityName,Event,ActivationRuleTreeId","EntityName == \"<Table>\"")
   + McpGetRuleTree(treeId)
4) F5 côté UI (config pure, pas de build)
```

### Masquer une section / un bouton
Identique avec `eventName="Section"` (cible `DynamicSection.Id`) ou `"DynamicButton"` (cible `DynamicButton.Id`).

### Filtre de liste déroulante (champ référence) — COMPLET
1. `McpCreateBusinessRule("Filter", entity, "Field", <DynamicField.Id du champ référence>, name, activationTreeJson, "")`
   — l'arbre s'évalue sur **la cible du lien** (ex. `entity.Building equals parentEntity.Building` filtre les
   équipements au bâtiment du ticket — modèle réel démo « Filter equipment by building »).
2. **Arbre de filtre** : `McpSetBusinessRuleExtras(ruleId, '{"actionTree": <RuleNodeDto>}')` →
   pose `ActionRuleTreeId` (le filtre lui-même ; l'activation conditionne QUAND il s'applique).

### Validation no-code — COMPLET
`McpCreateBusinessRule("Validation", entity, "Entity", <DynamicSettings.Id>, name, treeJson, "")` puis
`McpSetBusinessRuleExtras(ruleId, '{"validationMessageKey":"MaCleDeRessource","priority":10}')`.

### Affectation no-code (poser une valeur si condition)
1. `McpCreateBusinessRule("Affectation", entity, "Entity"|"Field", <cibleId>, name, activationTreeJson, "")`.
2. `McpSetBusinessRuleExtras(ruleId, '{"affectationSpec":{"source":1,"value":"entity.AutreChamp"}}')`
   — `source` : 0=LITERAL (valeur fixe), 1=MODEL (chemin de propriété), 2=CONTEXT (`global.*`),
   3=FUNCTION. ⚠️ C'est une **spec `ParameterValueDto`**, pas un arbre. ✅ validé en réel.

---

## 9. Lire l'existant (audit no-code via MCP)

- **Compter/lister par action** : les sous-classes sont **requêtables par leur nom** —
  `get_many_select("HideRule"|"ShowRule"|"ReadOnlyRule"|"MandatoryRule"|"ValidationRule"|"FilterRule",
  "Name,EntityName,Event,ViewName,ActivationRuleTreeId", "EntityName == \"X\"")`.
  (Sur le type de base `BusinessRule` : `Name,EntityName,Event,Type,Priority,ViewName` passent,
  mais PAS `ActivationRuleTreeId` — propriété des sous-classes.)
- **Lire la condition** : `McpGetRuleTree(<ActivationRuleTreeId>)`.
- **Résoudre la cible** : `Event=1` → `get_many_select("DynamicField","EntityName,PropertyName","Id == \"<EntityId>\"")`.
- **RuleSets réutilisables** : `get_many_select("RuleSet","Name,EntityName,Description")` ; arbre du
  RuleSet : `McpGetRuleTree(<RuleSet.Id>)` (convention `RuleTreeId = RuleSet.Id`).

---

## 10. Gotchas catalogués (no-code — issus de vrais essais)

- **`&&` dans un `where` MCP peut arriver HTML-encodé (`&amp;&amp;`) ⇒ `Syntax error ';'`** →
  utiliser **`and` / `or`** (Dynamic LINQ les accepte). ✅ constaté.
- **`RuleNode`/`RuleClause`/`ParameterValue` non requêtables** via `get_many_select` (pas `IEntity`) →
  passer par `McpGetRuleTree`/`RuleTreeService`. ✅ constaté.
- **`SaveOrReplace` régénère les ids des nœuds** (DeepCloneAndNormalize) — ne JAMAIS référencer un id
  de nœud ; seul le `ruleTreeId` (= la valeur posée sur `ActivationRuleTreeId`) est stable.
- **Littéraux booléens** observés stockés en string (`"value":"false"`) ; numériques en décimal.
  À l'écriture, les deux formes passent (coercition standard du moteur).
- **`Type` est posé par le constructeur** de la sous-classe — ne pas le setter à la main ;
  instancier la BONNE classe (`new HideRule()`…).
- **Roots sensibles à la casse** : `entity.X`, `parentEntity.X` — `Entity.X` ⇒ « Property 'Entity' not found ».
- **Suppression** : TOUJOURS `DeleteWithDependencies` (un `Delete` simple laisserait des arbres orphelins).
- **⚠️ Affectation ≠ arbre (incident vécu).** Poser un **RuleTree** sous `AffectationRuleTreeId` crée un
  **orphelin indestructible par la cascade** (`DeleteWithDependencies` appelle `DeleteAffectationSpec` =
  `DeleteParameterValue`, qui ne supprime pas des RuleNodes). Toujours passer par
  `McpSetBusinessRuleExtras{affectationSpec}` (→ `PersistAffectationSpec`). Récupération d'un orphelin :
  `McpDeleteRuleTree(treeId)`.
- **Gap appli (`BusinessRuleService.cs:74-78`)** : `GetAffectationId` ne matche QUE `AffectationRule` —
  la cascade ne nettoie PAS l'affectation d'une `AlertRule` (fuite potentielle, à nettoyer à la main).
- **Template vs override** : `ViewName=null` = règle de la table maître (héritée par les vues) ;
  une règle posée avec un `ViewName` précis ne s'applique qu'à cette vue (mécanisme `TemplateEntity`).
- Règles **config pures** : effet F5, pas de `McpBuild`.
