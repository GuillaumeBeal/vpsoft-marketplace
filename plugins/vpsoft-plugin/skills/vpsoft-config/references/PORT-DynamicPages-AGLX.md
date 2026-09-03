# Porter du code de DynamicPage entre instances — transport & AGLX

**Validé en conditions réelles le 2026-09-02** (portage de la page `FIB` de Vesta 10.4 vers
`FIB_INSPECTION` sur Vesta_Demo 10.5 : 4 pages, ~110 Ko de Razor/CSS/JS).

---

## 1. La règle d'or : QUI transporte les octets ?

Le goulot n'est jamais le helper serveur, c'est le **canal**. Un LLM n'est pas un canal binaire fiable.

| Route | Verdict |
|---|---|
| Export/import **AGLX** (natif VPSoft) | ✅ la plus complète — porte code + rôles + dossier + libellés (cf. §4) |
| Fichier local → **navigateur** → `McpUpdateDynamicPageCode` (texte brut) | ✅ **méthode préférée pour un push ciblé** (§2) |
| Fichier → navigateur → helper base64 + SHA-256 | ⚠️ correct mais une couche inutile (§3) |
| Chunks base64 **recopiés par un sous-agent** | ❌ **ANTI-PATTERN — ~70 min perdues, jamais abouti** |

> ⚠️ **L'anti-pattern en détail.** Faire recopier du base64 par un agent : il perd ou ajoute des
> caractères (−4 en fin de chunk, +12 au milieu…). Réduire la taille des chunks **ne converge pas** :
> 6000 → 2000 → 20 caractères, **138 appels** pour les seuls slots d'une page, sans jamais finir.
> **Si tu te vois réduire la taille des chunks, ARRÊTE et passe par le navigateur.**

---

## 2. ✅ Méthode préférée — `McpUpdateDynamicPageCode` depuis le navigateur

`McpUpdateDynamicPageCode(code, codeRazor, codeCss, codeJavascript)` existe déjà au catalogue et prend
le code en **chaînes brutes** (param vide = slot conservé). Aucun base64, aucun helper à créer :
c'est le navigateur qui construit la requête, donc `JSON.stringify` gère l'échappement nativement.

**Recette** (les octets font disque → navigateur → serveur, sans jamais passer par le LLM) :

1. Écrire le code adapté dans des fichiers locaux (`scratchpad/<page>.razor.cshtml`, `.css`, `.js`).
   Toujours `node --check` le JS avant.
2. Ouvrir une page VPSoft **authentifiée** (n'importe laquelle du même app) dans le navigateur.
3. Injecter un input de fichiers, puis y déposer les fichiers avec l'outil d'upload du navigateur :
   ```js
   let i = document.createElement('input');
   i.type = 'file'; i.id = 'claudeFileIn'; i.multiple = true;
   i.setAttribute('aria-label', 'claudeFileIn');   // le rend trouvable par `find`
   i.style.cssText = 'position:fixed;top:4px;left:4px;z-index:99999;background:#fff';
   document.body.appendChild(i);
   ```
4. Envoyer depuis la page :
   ```js
   const f = [...document.getElementById('claudeFileIn').files];
   const txt = async n => { const x = f.find(v => v.name === n); return x ? await x.text() : ''; };
   const resp = await VP.Functions.Invoke('McpUpdateDynamicPageCode',
       ['FIB_INSPECTION', await txt('FIB_INSPECTION.razor.cshtml'),
                          await txt('FIB_INSPECTION.css'),
                          await txt('FIB_INSPECTION.js')]);
   ```
5. **Retirer l'input** une fois fini (`document.getElementById('claudeFileIn').remove()`).
6. Vérifier : `McpRenderDynamicPage(code)` (compile le Razor sans navigateur) puis test visuel.

> ⚠️ **`window.VP` est `undefined`** alors que `VP` nu existe (binding global du bundle).
> Tester/appeler **`VP`** nu — sinon faux négatif (« VP absent » alors qu'il est là).
> Sur une **page pleine**, le bundle `VP` est chargé en bas : attendre `VP` par un poll si besoin.

> ⚠️ `DynamicPageCodeViewModel.GitFileBasePath` est `required` → toujours passer par
> `GetCode(id)` puis `SaveCode(vm)` (ce que fait `McpUpdateDynamicPageCode`), jamais un `new {}`.

---

## 3. Variante avec contrôle d'intégrité — `McpSetDynamicPageSlotB64`

Utile seulement si tu veux une **preuve cryptographique** du round-trip (ou si tu dois quand même
découper). Signature : `(code, slot[razor|css|js], b64, reset, finalize, gzip)` — accumule les chunks
base64 dans `CodeJavascript` (buffer), puis au `finalize` décode → écrit le slot cible → renvoie
`total` + **`sha256`** (hex des octets UTF-8). Comparer au `crypto.subtle.digest` local.

Gotchas de ce helper (tous vécus) :
- ⚠️ **`gzip` doit valoir `false`** : `System.IO.Compression.GZipStream` **n'est pas résolvable** en
  compilation à chaud des DynamicFunction → retour `null` **muet** (erreur Roslyn, absente d'AppLog).
- ⚠️ Un bloc **`using (var h = SHA256.Create()) { … }`** fait aussi échouer la compilation en silence
  → instancier sans `using` (`var alg = SHA256.Create();`).
- ⚠️ `total` = `string.Length` **UTF-16** ≠ octets UTF-8 dès qu'il y a accents/emoji → **se fier au
  `sha256`, jamais à `total`**.

---

## 4. Route native testée : export / import **AGLX**

Le vrai mécanisme VPSoft pour déplacer de la configuration entre instances.
Code source : `VPSoft.Services/AGLX/AglxMainService.cs`, modules par type dans
`VPSoft.Services/AGLX/DynamicCodeModule/` (dont `AglxDynamicPageService`).

### API
| Méthode (`sm.AglxMainService`) | Rôle |
|---|---|
| `GetExportViewModel()` | liste les modules exportables (`id` = **AglxId**, `code`, `name`) |
| `Export(ExportFormModel{ModuleIds, ImportScope})` | → `AglxExportJson` |
| `ExportAsFile(AglxExportJson)` | → `(nomFichier.aglx, dossier, Stream)` — un **ZIP** |
| `ProcessModules(IFormFile, importId, description, isFromPreview, out …)` | **preview du diff** → `AglxTreeDiffResult` |
| `Import(ImportFormModel, out AglxImport, out hasSomeLogs)` | applique |
| `GetBackUpFile(id)` / `GetLogsFile(id)` / `GetImportedFile(id)` / `GetHistory()` | traçabilité & rollback |

### Ce que l'AGLX porte pour une page (`AglxDynamicPageModel`)
`Code`, `IsIndependent`, **`CodeRazor` / `CodeCss` / `CodeJavascript`**, `CodeCompiled`,
`DynamicPageType`, `DynamicFolderAglxId`, **`RoleInModuleAglxIds`** (les rôles d'accès !),
`CultureParameters` (libellés localisés), fichiers CSS/JS/images, `IsMergeable`/`MergedChecksum`.
→ **Plus complet qu'un push de code** : rôles, dossier et libellés voyagent avec.

### Granularité (le point clé)
- **Export = par MODULE entier** (`ExportFormModel.ModuleIds`), pas par page.
- **Import = sélectif jusqu'à la propriété** :
  `ImportFormModel.ModuleTargets` → `AglxModuleTarget{AglxId, Type, SelectedTargets}` →
  `AglxTarget{Type, AglxId, SelectedProperties}`.
  → on exporte le module puis on **n'importe que les pages voulues**, voire seulement leur `CodeRazor`.
- `AglxImportScope` : `Unique=0`, `Selected=1` (défaut), `Global=2`.

### Mesures réelles (demo-vesta-10-5, module Portfolio)
```
McpAglxExportProbe("export", "d66d5977-ced6-49b6-b217-a8ceb52e8e85")
→ sourceVersion 10.5.18 | moduleCount 1 | jsonLength 8 365 294 (8,4 Mo)
  fibInspectionOccurrences 18 | Razor ✅ | CSS ✅ | JS ✅
```
**8,4 Mo pour un module** là où les 4 pages pèsent ~110 Ko (×75). L'export est donc lourd,
mais l'import sélectif rend l'opération précise.

### Gotchas AGLX
- ⚠️ **`ModuleIds` attend des `AglxId`, PAS les `Id` d'entité** (ex. Portfolio :
  AglxId `d66d5977-…` vs Id `f15e0ea8-…`). Les résoudre via `GetExportViewModel()`.
- ⚠️ **Aucun contrôle de version à l'import** : `AglxExportJson.SourceVersion` est **écrit mais jamais
  relu** dans tout l'AGLX (vérifié par grep sur les services/contrôleurs/contrats). Un import
  **10.4 → 10.5 n'est donc ni bloqué ni protégé** — prévoir la preview du diff et le backup.
- ⚠️ `Import`/`ProcessModules` attendent un **`IFormFile`** → c'est naturellement une opération
  **UI/navigateur**, pas un pur appel MCP. Une DynamicFunction peut faire l'`Export`, mais pour
  l'import il faut passer par l'écran AGLX (ou fabriquer un `IFormFile`, peu pratique).
- L'export crée un dossier temporaire côté serveur (`AglxHelper.CreateExportDirectory`).

### Quand choisir quoi
- **Portage complet d'un module / mise en prod d'une couche consultant** → **AGLX** (traçable, backup, rollback).
- **Pousser/patcher le code de quelques pages** → **`McpUpdateDynamicPageCode` via navigateur** (§2) : instantané, ciblé.

---

## 5. Piège de rendu à connaître (page pleine)

Une **page pleine ouverte par le menu** exécute son `<script>` **avant que le DOM soit prêt**.
Conséquence : un `document.getElementById('x').addEventListener(...)` **non gardé** lève
`Cannot read properties of null` et **annule TOUS les bindings suivants** de l'IIFE
→ symptôme trompeur : « les boutons ne font rien », parfois **sans erreur visible** en console
(l'exception a lieu au chargement, avant que l'outil de lecture console ne s'attache).

**Correctif** — envelopper l'init dans un boot qui attend le DOM *et* l'élément clé :
```js
(function () {
  'use strict';
  function __init() { /* … tout le code exécutable … */ }
  (function boot() {
    if (document.readyState === 'loading') { document.addEventListener('DOMContentLoaded', boot, { once: true }); return; }
    if (!document.getElementById('<element-cle>')) { setTimeout(boot, 50); return; }
    __init();
  })();
}());
```

## 6. Adaptations de champs 10.4 → 10.5 (rencontrées sur la FIB)

| 10.4 | 10.5 | Note |
|---|---|---|
| `Organization.FullAdress` | `AddressInLine` (souvent vide) sinon `AddressLine1`+`PostCode`+`City` | à concaténer |
| `Area` (décimal nullable) | `Surface` | ⚠️ vérifier nullable : `(orga.Surface ?? 0) > 0` puis `.Value` |
| `BuildDate` (DateTime) | `DeliveryYear` (**int**, année seule) | changement de type |
| `Persona.Function` | *absent* → `Team?.Name` | pas d'équivalent direct |
| `Theme`/`ThemeExtension` `.Color` / `.Icon` | **absents des deux** | prévoir couleur neutre / pas d'icône |
| `CollectionExtension.Icon` | *absent* | — |

> ⚠️ **Retirer un élément d'un `display:grid` casse le mapping des colonnes.** En supprimant l'icône
> de thème, la colonne `1fr` (destinée au titre) s'est décalée sur une autre cellule → titre rendu
> **mot par mot** sur 36 px. Toujours réaligner `grid-template-columns` (y compris dans les media
> queries) après avoir supprimé un enfant de grille.
