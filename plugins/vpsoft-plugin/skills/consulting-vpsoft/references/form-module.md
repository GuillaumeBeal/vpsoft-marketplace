# Module Form — Questionnaires, Campagnes et Formules

Reference complete du module Form de VPSoft. Ce module permet de creer des questionnaires configurables, les deployer en campagnes, et calculer des scores via des formules.

Code source C# : `VPSoft.Domain/Models/Forms/` (dans le projet VPSoft.Domain)

## Architecture du module

```
Campaign (Campagne de deploiement)
│ ├── Forms[] (Formulaires a deployer)
│ ├── Organizations[] (Perimetre organisationnel)
│ ├── Frequency, DeploymentStatus, DeploymentType
│ └── CampaignOccurrence[] (⚠ non accessible via API V2)
│
├── Questionary (Instance deployee d'un formulaire)
│   ├── Version (0=template, >=1=version)
│   ├── Campaign (FK)
│   ├── Organization, Theme (perimetre)
│   └── Users[] (utilisateurs assignes)
│
Form (Formulaire)
│ ├── Version, Prefix, Code, Name
│ ├── FormWorkflow[] (workflow de validation)
│ ├── DeployableEntities[] (entites deployables)
│ │
│ └── Section[] (Sections)
│     ├── Code, OrderBy, Weight, Level
│     ├── SectionChildrenList[] (sous-sections)
│     ├── IsTable, QuestionTable[] (sections tabulaires)
│     │
│     ├── FormQuestion[] (Questions)
│     │   ├── QuestionType, IsRequired, Weight
│     │   ├── Formula, DefaultValue, ConditionalDisplay
│     │   ├── FormAnswer[] (Reponses possibles)
│     │   │   ├── ResponseWeight, Color, IsDefault
│     │   │   └── ResponsePreset (FK vers ResponseList)
│     │   ├── ResponsePresetCategory (FK vers ResponseListCategory)
│     │   └── QuestionDisplayCondition[] (conditions affichage)
│     │
│     └── FormFormula[] (Formules de calcul)
│         ├── Formula (expression)
│         └── FormulaInterval[] (intervalles de classe)
│
└── UserResponse (Reponses utilisateur)
    ├── Response, DigitalNumber, OpenQuestionText
    ├── Question (FK), Questionary (FK)
    ├── QuestionWeight, ResponseWeight
    └── Answers[] (reponses selectionnees)
```

## Entites principales

### Form — Formulaire

| Champ | Description |
|-------|-------------|
| Code | Code unique du formulaire |
| Name | Nom localise (multi-culture) |
| Description | Description localisee |
| Version | Numero de version (incremente a la publication) |
| Prefix | Prefixe de code |
| ParentCode | Code du formulaire parent (template) |
| TemplateId | ID du template d'origine |
| IsEnableValidationRulesOnlyForWorkflow | Validation uniquement via workflow |

Relations : Sections[], FormWorkflows[], Themes[], Organizations[], DeployableEntities[]

### Section — Section de formulaire

| Champ | Description |
|-------|-------------|
| Code | Code unique |
| Reference | Reference externe |
| OrderBy | Ordre d'affichage |
| Weight | Poids dans le calcul |
| Level | Niveau de profondeur |
| Name | Nom localise |
| IsTable | Section en mode tableau |
| AllowCopying | Autoriser la duplication |
| HideInQuestionary | Masquer dans le questionnaire |
| Constant | Valeur constante |

Relations : Form, FormQuestions[], Formulas[], QuestionTable[], SectionChildrenList[], ParentSection

### FormQuestion — Question

| Champ | Description |
|-------|-------------|
| Code | Code unique |
| Question | Texte de la question (localise) |
| Description | Description/aide (localisee) |
| Order | Ordre d'affichage |
| Weight | Poids dans le calcul |
| Reference | Reference externe |
| QuestionType | Type de question (enum FormQuestionType) |
| IsRequired | Question obligatoire |
| RegEx | Expression reguliere de validation |
| RegExValidationText | Message de validation |
| DecimalLength | Precision decimale |
| ConditionalDisplay | Affichage conditionnel |
| Formula | Formule de calcul sur la question |
| DefaultValue | Valeur par defaut |
| DefaultMessage | Message par defaut |
| ActivateComment | Activer les commentaires |
| ActivateAttachment | Activer les pieces jointes |
| ActivateQuestionValidation | Activer la validation |
| ActivateWithResponseMultiple | Reponses multiples |
| ShowDescription | Afficher la description |
| NotTakenInTheCalculation | Exclure du calcul |
| JavaScriptQuestion | Code JS sur la question |
| MaxResponseWeight | Poids max des reponses |
| DisplayInSelectList | Afficher en liste deroulante |
| DisplayHelpFromSideMenu | Aide dans le menu lateral |

Relations : Section, Form, Answers[], ResponsePresetCategory, Measure, Keywords[], AttachableEntities[]

Champs de comparaison de formules : LessOrEqual, Less, SuperiorOrEqual, Superior, Equal

### FormAnswer — Reponse

⚠ FormAnswer n'a pas de champ EntityState — omettre le filtre EntityState dans WHERE.

| Champ | Description |
|-------|-------------|
| Code | Code unique |
| Name | Texte de la reponse (localise) |
| Answer | Reponse (pour questions vrai/faux) |
| Order | Ordre d'affichage |
| IsDefault | Reponse par defaut |
| ResponseWeight | Poids de la reponse |
| NotTakenInTheCalculation | Exclure du calcul |
| Opacity | Opacite visuelle |
| Color | Couleur associee |
| DisplayInSelectList | Afficher en liste |

Relations : Question, ResponsePreset

### FormFormula — Formule de calcul

| Champ | Description |
|-------|-------------|
| Code | Code unique |
| Formula | Expression de calcul |
| LocalizedDescription | Description localisee |
| IsDisplayQuestionary | Afficher dans le questionnaire |

Relations : Section (optionnel), Form (optionnel)

### FormulaClass — Classe de formule

| Champ | Description |
|-------|-------------|
| Code | Code unique |
| Name | Nom |
| Html | Rendu HTML |
| Preview | Apercu |

### FormulaInterval — Intervalle de formule

| Champ | Description |
|-------|-------------|
| LeftOperator | Operateur gauche |
| RightOperator | Operateur droit |
| LeftValue | Valeur gauche |
| RightValue | Valeur droite |
| OrderBy | Ordre |

Relations : FormulaClass, FormFormula

## Deploiement et Campagnes

### Campaign — Campagne

⚠ Campaign n'a pas de champ Code accessible via API V2 — utiliser Name comme identifiant.

| Champ | Description |
|-------|-------------|
| Name | Nom de la campagne (identifiant principal) |
| Objective | Objectif de la campagne |
| Frequency | Frequence (Bimonthly, Monthly, Quarterly, HalfYearly, Yearly, Weekly) |
| DeploymentStatus | Statut (Draft, ToDeploy, Deployed, Finished) |
| DeploymentType | Type (InAdvance, InTime) |
| DeploymentDate | Date de deploiement |
| StartReportingDate | Debut de reporting |
| EndReportingDate | Fin de reporting |
| StartCollectingDate | Debut de collecte |
| EndCollectingDate | Fin de collecte |
| Hits | Nombre de hits prevus |
| CurrentHit | Hit courant |
| EditableDays | Jours d'editabilite |
| ActivateProcess | Activer les processus |
| ActivatePreFilling | Activer le pre-remplissage |
| EntityName | Entite cible |

Relations : Forms[], Organizations[], EntitiesToDeploy[], PreFillCampaign, Theme

### Questionary — Questionnaire deploye

| Champ | Description |
|-------|-------------|
| Code | Code unique |
| Name | Nom |
| Description | Description |
| Version | 0=template, >=1=version deployee |
| InternalName | Nom interne |
| Objective | Objectif |
| Reference | Reference |
| QuestionaryType | Type (campagne ou entite) |
| QuestionsProgressionPercentage | Pourcentage de progression |
| IsEditable | Modifiable |
| StartDate | Date de debut |
| EndDate | Date de fin |
| ObtainingDate | Date d'obtention |
| ExpirationDate | Date d'expiration |
| FormDynamicEntityName | Entite liee (nom) |
| FormDynamicEntityId | Entite liee (GUID) |

Relations : Campaign, Organization, Theme, AuditorOwner (User), Users[]

### DeployableEntity — Entite deployable

| Champ | Description |
|-------|-------------|
| TechnicalName | Nom technique de l'entite |
| BusinessName | Nom metier |

## Reponses utilisateur

### UserResponse — Reponse

⚠ UserResponse n'a pas de champ Code accessible — utiliser Question.Code et Questionary.Code comme cles.
⚠ QuestionWeight et ResponseWeight ne sont pas accessibles via API V2.

| Champ | Description |
|-------|-------------|
| Response | Reponse textuelle |
| OpenQuestionText | Texte libre |
| OpenQuestionCorrection | Correction |
| DigitalNumber | Valeur numerique |
| TimeValue | Valeur temporelle |
| Version | Version de la reponse |
| IsDefaultResponse | Reponse par defaut |
| IsValidatedQuestion | Question validee |
| NotTakenInTheCalculation | Exclure du calcul |
| EntryResponse | Reponse saisie |
| FormulaCodes | Codes de formules |
| MeasureSymbol | Symbole d'unite |

Relations : Question, Form, Questionary, FormDuplicatedSection, File, Answers[], DynamicMeasure

### UserResponseHistorical

Historique des reponses precedentes (meme structure que UserResponse).

## Workflow formulaire

### FormWorkflow

| Champ | Description |
|-------|-------------|
| Form | Formulaire (FK) |
| EntityWorkflowStatus | Statut workflow (FK) |

### FormWorkflowByRole

| Champ | Description |
|-------|-------------|
| FormWorkflow | Workflow (FK) |
| Role | Role (FK) |
| UserPropertyName | Propriete utilisateur |
| Validate | Peut valider |
| Reject | Peut rejeter |
| CanGoFirst | Peut aller au debut |
| CanGoLast | Peut aller a la fin |
| CanVersion | Peut creer une version |

### FormWorkflowActionBySection

| Champ | Description |
|-------|-------------|
| FormWorkflowByRole | Role workflow (FK) |
| Section | Section (FK) |
| SectionAction | Action sur la section (enum) |

## Questions tabulaires

Les sections en mode tableau utilisent des entites dediees :

- **QuestionTable** : Matrice question/ligne/colonne (ML=row, MC=col)
- **TableRow / TableRowCategory** : Lignes et categories
- **TableColumn / TableColumnCategory** : Colonnes et categories
- **QuestionTableValue** : Valeurs dans la matrice

## Listes de reponses preedefinies

- **ResponseListCategory** : Categorie de liste (Code, Name, Description)
- **ResponseList** : Element de liste (Code, LocalizedName, ResponseWeight, NotTakenInTheCalculation)

## Conditions d'affichage

### QuestionDisplayCondition

⚠ QuestionDisplayCondition n'a pas de champ Code accessible — utiliser Name et Question.Code.

| Champ | Description |
|-------|-------------|
| Name | Nom de la condition |
| Question | Question source (FK) |
| Answers[] | Reponses declencheuses |
| QuestionsToDisplay[] | Questions a afficher |

## Syntaxe des formules et expressions

### Champ "Formula" sur FormQuestion

Le champ Formula stocke une **expression C#** evaluee a l'execution via **Microsoft Roslyn CSharp Scripting** (`CSharpEngineService.EvaluateAsync()`). Les resultats sont caches par hash MD5 pour la performance.

**Syntaxe supportee :**
- Arithmetique : `5 + 3`, `10 / 2`, `(a + b) * c`
- Comparaisons : `x > 5`, `y == 10`, `z != 0`
- Logique booleenne : `x > 5 && y < 10`, `a || b`
- Variables : remplacees par les valeurs reelles des reponses au moment de l'evaluation

**Utilisation avec FormulaInterval :**
Les formules sont souvent combinees avec des intervalles (`FormulaInterval`) pour classer les resultats :
```
Expression generee : "{result} >= {LeftValue} && {result} <= {RightValue}"
Exemple concret   : "5.2 >= 1 && 5.2 <= 10"
```

Operateurs disponibles dans FormulaInterval : `>`, `<`, `>=`, `<=`, `==`, `!=`

**Attention** : Les virgules decimales sont automatiquement remplacees par des points (`Replace(',', '.')`) pour l'independance culturelle.

### Champ "ConditionalDisplay" sur FormQuestion

Le champ ConditionalDisplay est un **flag boolean** (`true`/`false`). Il active ou desactive la visibilite conditionnelle de la question.

Quand `ConditionalDisplay = true`, la logique de visibilite est geree par l'entite **QuestionDisplayCondition** (pas par une expression dans ce champ).

**Fonctionnement :**
1. `ConditionalDisplay = true` sur la question cible
2. Creer une `QuestionDisplayCondition` qui lie :
   - `Question` : la question source (celle dont la reponse declenche l'affichage)
   - `Answers[]` : les reponses declencheuses
   - `QuestionsToDisplay[]` : les questions a afficher/masquer

**Requete pour lire les conditions :**
```
get_many_select("QuestionDisplayCondition",
  "Name, Question.Code",
  where='Question.Form.Code == "MON_FORM"', take=50)
```

⚠ Dans le fichier d'import FormQuestion, la colonne "Affichage conditionnel" attend un **boolean** (True/False), pas une expression. Les conditions elles-memes doivent etre configurees separement (pas via import).

### Champs de comparaison sur FormQuestion

Cinq champs supplementaires stockent des **expressions C#** pour des comparaisons de valeurs :

| Champ | Propriete C# | Utilisation |
|-------|-------------|-------------|
| LessOrEqual | LessOrEqual | Expression "inferieur ou egal" |
| Less | Less | Expression "strictement inferieur" |
| SuperiorOrEqual | SuperiorOrEqual | Expression "superieur ou egal" |
| Superior | Superior | Expression "strictement superieur" |
| Equal | Equal | Expression "egal" |

Ces champs suivent la meme syntaxe C# que Formula (evaluees par Roslyn).

### Champ "JavaScriptQuestion" sur FormQuestion

Code JavaScript execute cote frontend pour la logique custom. Active via `AppConfig.ActivateDisplayJavaScriptQuestion`. Stocke en `nvarchar(max)`.

---

## Notes sur les entites Form (framework)

Les entites du module Form sont des entites framework (0 champs via `get_entity_fields()`).
Elles sont neanmoins requetables via `get_many_select()` avec les noms de champs C# du domaine.

**Specificites** :
- `FormAnswer` : pas de champ EntityState — omettre le filtre EntityState dans WHERE
- `Campaign` : pas de champ Code — utiliser Name comme identifiant
- `UserResponse` : pas de champ Code — utiliser Response, DigitalNumber, Question.Code
- `QuestionDisplayCondition` : pas de champ Code — utiliser Name, Question.Code
- `order_by` multi-colonne (ex: `"Level, OrderBy"`) ne fonctionne pas — utiliser un seul champ
- `FormulaInterval`, `FormulaClass` : peuvent avoir 0 enregistrements sur certaines instances

## Import/Export — Production de fichiers d'import

Le module Form dispose d'un systeme d'import/export Excel (.xlsx) pour 3 entites : **Section**, **FormQuestion**, **FormAnswer**. Ce systeme permet de charger massivement la structure d'un formulaire a partir de fichiers Excel.

**Cas d'usage principal** : Un client fournit un fichier chaotique (Word, Excel non structure, PDF) decrivant un questionnaire. Le consultant doit transformer ces donnees sources en 3 fichiers Excel conformes au format d'import VPSoft, en respectant les regles de coherence et d'unicite.

### Contraintes generales

1. **Format** : Excel .xlsx (bibliotheque NPOI)
2. **Template uniquement** : L'import ne fonctionne que sur les formulaires templates (`Form.TemplateId == Guid.Empty`). Impossible d'importer dans un formulaire versionne/deploye.
3. **Unicite des codes** :
   - Chaque Code doit etre unique **dans le fichier** (pas de doublons entre lignes)
   - Chaque Code doit etre unique **en base de donnees** (pas de collision avec un code existant)
   - La verification est faite via `GetUniqueCode*` avant import
4. **Ordre d'import** : Sections d'abord, puis Questions, puis Reponses (les FK doivent exister)
5. **Post-import** : Apres import des Sections, la stored procedure `spInitTreeSectionSingle` reconstruit l'arborescence
6. **Colonnes multi-langue** : Les champs de type MultiCulture (Name, Question, Description, Comment) generent **une colonne par langue active**. Exemple avec 3 langues : `Name[fr-FR]`, `Name[en-US]`, `Name[es-ES]`. Le nombre de colonnes dans le fichier depend donc du nombre de cultures actives.
7. **Decouverte des langues actives** :
   ```
   get_many_select("Culture", "CultureCode, CultureCodeShort, IsDefault, IsLocal",
     where='EntityState == "Active"', take=20)
   ```
   L'entite `Culture` contient : `CultureCode` (ex: "fr-FR"), `CultureCodeShort` (ex: "fr"), `IsDefault` (langue par defaut), `IsLocal` (culture locale/base).

### Formatage du fichier xlsx produit

Quand la skill produit un fichier .xlsx d'import, appliquer ces regles de formatage :

- **Colonnes obligatoires** : en-tete avec fond **orange** (`#FF8C00`) et texte en gras
- **Colonnes optionnelles** : en-tete avec fond gris clair (`#D9D9D9`)
- **Ligne d'en-tete** : premiere ligne figee (freeze pane)
- **Colonnes multi-langue** : grouper visuellement les colonnes d'une meme propriete (ex: Name[fr-FR], Name[en-US] cote a cote)
- **Largeur auto** : ajuster la largeur des colonnes au contenu

### Fichier 1 — Section (import via SectionController)

⚠ **Contrainte mono-formulaire** : Toutes les lignes du fichier doivent avoir le meme `Form_Code`. Le systeme verifie cette contrainte avant import.

| Colonne | Propriete C# | Requis | Description |
|---------|-------------|--------|-------------|
| Id | Id | Non | GUID (genere si vide) |
| Name[fr-FR] | Name | Oui | Nom localise (multi-culture : une colonne par langue, ex: Name[en-US]) |
| Code | Code | **Oui** | Code unique — unicite fichier + DB |
| Formulaire_Code | Form | **Oui** | Code du formulaire cible — **identique pour toutes les lignes** |
| Position | OrderBy | Non | Ordre d'affichage (entier) |
| Ponderation | Weight | Non | Poids dans le calcul (decimal) |
| Constant | Constant | Non | Valeur constante |
| Level | Level | Non | Niveau de profondeur (0 = racine) |
| Autoriser la duplication | AllowCopying | Non | Boolean (true/false) |
| Section Tableau | IsTable | **Oui** | Boolean — section en mode tableau |
| Form Categories-Code | FormCategories | Non | Codes categories separes par `;` |
| FormTypes-Code | FormTypes | Non | Codes types separes par `;` |
| Section Parent_Code | ParentSection | **Oui** | Code de la section parente (vide si racine) |
| EntityState | EntityState | Non | 0=Draft, 1=Active (defaut: Active) |

**Regles de validation** :
- `Code` : verifie unique dans le fichier ET en base via `GetUniqueCodeSection()`
- `Form_Code` : doit etre un formulaire existant ET template
- `ParentSection_Code` : si renseigne, doit correspondre a un Code present dans le fichier ou en base
- Collections (`FormCategories`, `FormTypes`) : codes separes par `;`, resolus par Code en base

### Fichier 2 — FormQuestion (import via FormQuestionController)

| Colonne | Propriete C# | Requis | Description |
|---------|-------------|--------|-------------|
| Id | Id | Non | GUID |
| Code | Code | **Oui** | Code unique — unicite fichier + DB |
| Reference | Reference | Non | Reference externe |
| Ordre | Order | Non | Ordre d'affichage (entier) |
| Description[fr-FR] | Description | Non | Description localisee (multi-culture). Auto-encapsulee dans `<p>` si pas deja HTML |
| Question[fr-FR] | Question | **Oui** | Texte de la question (multi-culture). Auto-encapsule dans `<p>` si pas deja HTML |
| Comment[fr-FR] | DefaultMessage | Non | Message/commentaire par defaut (multi-culture) |
| Type de question | QuestionType | Non | Nom du type (doit exister comme entite `FormQuestionType`) |
| Section_Code | Section | Non | Code de la section parente |
| Categorie de reponse_Code | ResponsePresetCategory | Non | Code categorie. **Auto-creee si inexistante** (INSERT SQL direct) |
| Ponderation | Weight | Non | Poids dans le calcul |
| Obligatoire | IsRequired | Non | Boolean |
| Activer commentaire | ActivateComment | Non | Boolean |
| Activer piece jointe | ActivateAttachment | Non | Boolean |
| Activer reponse multiple | ActivateWithResponseMultiple | Non | Boolean |
| Afficher description | ShowDescription | Non | Boolean |
| Non pris dans le calcul | NotTakenInTheCalculation | Non | Boolean |
| Afficher en liste deroulante | DisplayInSelectList | Non | Boolean |
| Activer validation | ActivateQuestionValidation | Non | Boolean |
| Expression reguliere | RegEx | Non | RegEx de validation |
| Message validation | RegExValidationText | Non | Message si RegEx echoue |
| Precision decimale | DecimalLength | Non | Nombre de decimales |
| Valeur par defaut | DefaultValue | Non | Valeur par defaut |
| Formule | Formula | Non | Expression de calcul |
| Affichage conditionnel | ConditionalDisplay | Non | Expression conditionnelle |
| Poids max reponse | MaxResponseWeight | Non | Decimal |
| Aide menu lateral | DisplayHelpFromSideMenu | Non | Boolean |
| Entites attachables | AttachableEntities | Non | Noms techniques separes par `;` (resolution par `TechnicalName` et non par Code) |
| Upload fichiers | FileUpload | Non | Noms separes par `;` (resolution par `Name`) |
| EntityState | EntityState | Non | Defaut: Active |

**Regles de validation** :
- `Code` : unicite fichier + DB via `GetUniqueCodeFormQuestion()`
- `QuestionType` : doit exister comme entite `FormQuestionType` en base
- `Question` et `Description` : auto-encapsules dans `<p>...</p>` s'ils ne commencent pas par `<` (detection HTML)
- `ResponsePresetCategory_Code` : si le code n'existe pas en base, **auto-creation via raw SQL INSERT** dans `ResponsePresetCategory`
- `AttachableEntities` : resolution par `TechnicalName` (pas par Code), separateur `;`
- `FileUpload` : resolution par `Name` (pas par Code), separateur `;`

### Fichier 3 — FormAnswer (import via FormQuestionController)

⚠ **FormAnswer n'a pas de champ EntityState** : le champ EntityState dans le fichier d'import est ignore.

| Colonne | Propriete C# | Requis | Description |
|---------|-------------|--------|-------------|
| Id | Id | Non | GUID |
| Question_Code | Question | Non | Code de la question parente |
| Code | Code | Non | Code de la reponse |
| Name[fr-FR] | Name | **Oui** | Texte de la reponse (multi-culture) |
| Taille maximum | MaxSize | Non | Entier |
| Une formule | Formula | Non | Expression de formule |
| Ordre | Order | Non | Ordre d'affichage |
| EntityState | EntityState | Non | **Ignore** (FormAnswer n'a pas d'EntityState) |
| Reponses predefinie_Code | ResponsePreset | Non | Code de la ResponseList. **Auto-creee si inexistante** |
| ResponseWeight | ResponseWeight | Non | Poids de la reponse (decimal) |
| Type de liste de reponses_Code | ResponseListCategory | Non | Code categorie de liste |

**Regles de validation** :
- `Question_Code` : doit correspondre a une question du formulaire template
- `ResponsePreset_Code` (ResponseList) : si le code n'existe pas, **auto-creation** via `ImportExportHelper.PostCommitActionForEntities()` — creation d'une entite `ResponseList` avec le code fourni
- `ResponseListCategory_Code` : la categorie doit exister (pas d'auto-creation pour les categories de listes ici, contrairement a `ResponsePresetCategory` sur FormQuestion)

### Pipeline d'import complet

```
1. Upload du fichier .xlsx
2. Lecture des lignes via NPOI (GenericImport)
3. Verification globale :
   - Template uniquement (Form.TemplateId == Guid.Empty)
   - Section : toutes les lignes ont le meme Form_Code
4. Pour chaque ligne :
   a. Generation ou validation du Code (unicite fichier + DB)
   b. Resolution des FK par Code (Form_Code → Form.Id, Section_Code → Section.Id, etc.)
   c. Traitement special Question/Description → wrapping <p> si pas HTML
   d. Traitement des collections (;-separated → liste d'entites)
   e. Auto-creation si necessaire (ResponsePresetCategory, ResponseList)
5. Import en base (Create via service layer)
6. Post-import :
   - Sections : spInitTreeSectionSingle (reconstruction arborescence)
   - Questions : mise a jour des versions
```

### Guide de transformation : fichier client → fichiers d'import

Quand un client fournit un fichier chaotique, suivre cette methodologie :

```
0. DECOUVRIR les langues actives de l'application :
   get_many_select("Culture", "CultureCode, IsDefault",
     where='EntityState == "Active"', take=20)
   → Determine le nombre de colonnes multi-langue dans chaque fichier
   → Exemple : si 2 cultures (fr-FR, en-US), Name donne 2 colonnes : Name[fr-FR], Name[en-US]

1. IDENTIFIER le formulaire cible :
   - Quel Form_Code ? (get_many_select sur Form pour trouver le code)
   - Est-ce un template ? (Version == 0 ou TemplateId == Guid.Empty)

2. EXTRAIRE la structure du fichier source :
   - Sections/chapitres → lignes du fichier Section
   - Questions → lignes du fichier FormQuestion
   - Reponses possibles → lignes du fichier FormAnswer

3. GENERER les codes uniques :
   - Convention : prefixe formulaire + incrementiel (ex: ESRS_S1_SEC_001, ESRS_S1_Q_001)
   - Verifier l'unicite via get_many_select sur les codes existants

4. CONSTRUIRE l'arborescence des sections :
   - Definir les Level (0 = racine, 1 = sous-section, etc.)
   - Renseigner ParentSection_Code pour chaque sous-section
   - Ordonner via OrderBy (Position)

5. ASSOCIER les questions aux sections :
   - Chaque question a un Section_Code
   - Ordonner via Order (Ordre)
   - Determiner le QuestionType (doit exister en base)

6. DEFINIR les reponses :
   - Chaque reponse a un Question_Code
   - Poids (ResponseWeight) si calcul de score
   - Ou utiliser des listes preedefinies (ResponsePresetCategory_Code)

7. VALIDER avant import :
   - Pas de doublons de Code dans chaque fichier
   - Tous les FK references (Section_Code, Question_Code) sont coherents
   - Form_Code identique sur toutes les sections
   - Les QuestionType existent en base
```

### Validation pre-import — Checklist avant upload

Avant d'uploader un fichier d'import, verifier systematiquement ces points. Produire un rapport de validation et corriger les erreurs avant import.

**Validations communes aux 3 fichiers :**
```
□ Format .xlsx (pas .xls, pas .csv)
□ Ligne d'en-tete presente avec les noms de colonnes exacts
□ Pas de lignes vides intercalees
□ Pas de colonnes en double
□ Nombre de colonnes conforme au schema attendu (>= 5 minimum)
```

**Fichier Section :**
```
□ Tous les Code sont non-vides et uniques dans le fichier
□ Tous les Code sont absents de la base (verifier via get_many_select)
□ Formulaire_Code identique sur TOUTES les lignes
□ Formulaire_Code correspond a un Form existant ET template (Version 0 / TemplateId vide)
□ Section Parent_Code : chaque reference pointe vers un Code present dans le fichier OU en base
□ Pas de reference circulaire dans l'arborescence parent-enfant
□ Level coherent avec la hierarchie (racine=0, enfant de racine=1, etc.)
□ Position (OrderBy) : valeurs numeriques, pas de doublons au meme niveau
□ IsTable : boolean valide (True/False)
□ Collections (FormCategories, FormTypes) : chaque code separe par ";" existe en base
```

**Fichier FormQuestion :**
```
□ Tous les Code sont non-vides et uniques dans le fichier
□ Tous les Code sont absents de la base
□ Section_Code : chaque reference pointe vers une Section existante (fichier ou base)
□ Type de question : chaque valeur correspond a un FormQuestionType existant en base
□ Question[lang] : au moins une langue remplie pour chaque ligne
□ Champs boolean (Obligatoire, Activer commentaire, etc.) : valeurs True/False
□ Precision decimale : entier >= 0 si renseigne
□ Poids (Ponderation) : decimal >= 0 si renseigne
□ Expression reguliere : syntaxe regex valide si renseignee
□ Entites attachables : chaque TechnicalName separe par ";" existe en base
□ ResponsePresetCategory_Code : si renseigne, noter qu'il sera auto-cree si absent
```

**Fichier FormAnswer :**
```
□ Question_Code : chaque reference pointe vers une FormQuestion template (pas une version)
□ Name[lang] : au moins une langue remplie pour chaque ligne
□ ResponseWeight : decimal si renseigne
□ Ordre : entier, pas de doublons pour une meme question
□ ResponseListCategory_Code : si renseigne, doit exister en base (pas d'auto-creation ici)
□ ResponsePreset_Code : si renseigne, noter qu'il sera auto-cree si absent
```

**Verification croisee entre les 3 fichiers :**
```
□ Chaque Section_Code reference dans FormQuestion existe dans le fichier Section
□ Chaque Question_Code reference dans FormAnswer existe dans le fichier FormQuestion
□ Les ResponsePresetCategory_Code sont coherents entre FormQuestion et FormAnswer
□ L'ordre d'import est respecte : Section → Question → Answer
```

**Requetes MCP pour verifier les collisions de codes :**
```
# Codes sections existants
get_many_select("Section", "Code",
  where='Form.Code == "MON_FORM" && EntityState == "Active"', take=500)

# Codes questions existants
get_many_select("FormQuestion", "Code",
  where='Form.Code == "MON_FORM" && EntityState == "Active"', take=500)

# Types de questions valides
get_many_select("FormQuestionType", "Code, Name",
  where='EntityState == "Active"', take=50)

# Categories de reponses existantes
get_many_select("ResponseListCategory", "Code, Name",
  where='EntityState == "Active"', take=100)

# Formulaire cible est bien un template ?
get_many_select("Form", "Code, Name, Version, TemplateId",
  where='Code == "MON_FORM" && EntityState == "Active"')
# → TemplateId doit etre "00000000-0000-0000-0000-000000000000"
```

### Types de questions disponibles (FormQuestionType)

Pour connaitre les types de questions disponibles sur une instance :
```
get_many_select("FormQuestionType", "Code, Name",
  where='EntityState == "Active"', take=50)
```

Types courants : OpenQuestion, UniqueChoice, MultipleChoice, YesNo, Numeric, Date, Rating, File, Table, Formula, Measure.

### Guide des erreurs d'import

Toutes les erreurs d'import levent une `ImportExportException`. Le message inclut le numero de ligne fautive (`[Line n°X]`).

**Erreurs de structure du fichier :**

| Erreur | Cause | Solution |
|--------|-------|----------|
| `Files aren't valid, authorized extensions : {0}` | Extension fichier non supportee | Utiliser .xlsx (pas .xls, .csv) |
| `The number of columns is invalid` | Moins de 5 colonnes dans le fichier | Verifier le schema du fichier (en-tetes) |
| `Invalid number of required columns in the imported file` | Nombre de colonnes ne correspond pas au template | Comparer les en-tetes avec le schema attendu |
| `The number or the column's name are not valid` | Noms d'en-tete incorrects | Verifier l'orthographe exacte des en-tetes |
| `No db column description found at position n°{0}` | Colonne a une position non mappee | Supprimer les colonnes en trop |
| `Duplicate column headers` | Colonnes en double dans l'en-tete | Supprimer les colonnes dupliquees |

**Erreurs de codes et references :**

| Erreur | Cause | Solution |
|--------|-------|----------|
| `the value is not unique` | Code deja existant en base | Verifier les codes existants via `get_many_select` et renommer |
| `No reference found for "{0}" on property "{1}" with value "{2}"` | FK pointe vers un code inexistant | Verifier que le code reference existe en base ou dans un fichier precedent |
| `No entity found with primary key {0} in table {1}` | GUID inexistant (mode update) | Verifier l'ID ou passer en mode creation |
| `No valid GUID provided {0}` | Format GUID invalide | Laisser la colonne Id vide (auto-genere) |

**Erreurs specifiques au module Form :**

| Erreur | Cause | Solution |
|--------|-------|----------|
| `You must link a response list category on the question` | FormAnswer a un ResponsePreset mais la question n'a pas de ResponsePresetCategory | Ajouter une ResponsePresetCategory_Code sur la FormQuestion |
| `You must link the answer to a question from a template, not a version` | FormAnswer liee a une question versionnee | Lier au template (Question.TemplateId == Guid.Empty) |
| `This import can be use only for create` | Tentative de modification via un import creation-only | Utiliser un template d'import adapte |
| `Error during import in dynamic form step {0}: {1}` | Erreur dans le code injecte (PostCreate, etc.) | Verifier le code dynamique sur l'entite concernee |

**Erreurs de validation de champs :**

| Erreur | Cause | Solution |
|--------|-------|----------|
| `the value exceed the maximum length of {0}` | Texte trop long | Raccourcir la valeur |
| `the value is not match the regular expression "{0}"` | Valeur ne respecte pas le RegEx du champ | Corriger la valeur selon le pattern attendu |
| `the value is not a valid date` | Format de date invalide | Utiliser le format ISO : `2024-01-15T00:00:00` |
| `the value is not a valid email` | Email mal formate | Corriger le format email |
| `Max entities exceeded, it's necessary to filter` | Trop de lignes dans le fichier | Decouper le fichier en plusieurs lots |

**Diagnostic apres erreur :**
```
# Verifier les erreurs applicatives recentes
get_app_errors(take=10)

# Verifier l'audit pour voir ce qui a ete importe partiellement
get_audit_history(entity_name="Section", take=20)
get_audit_history(entity_name="FormQuestion", take=20)
```

### Workflow Export → Transform → Re-import

Cas d'usage : modifier un formulaire existant en l'exportant, le transformant dans Excel, puis en re-important les modifications.

⚠ **Limitation critique** : L'import ne fonctionne que sur les **templates** (Version 0). Pour modifier un formulaire deploye, il faut modifier le template d'origine puis redeployer.

```
PHASE 1 — EXPORT (extraction via MCP)

1. Identifier le formulaire template :
   get_many_select("Form", "Code, Name, Version, TemplateId",
     where='Code == "MON_FORM" && EntityState == "Active"')
   # Verifier : TemplateId == "00000000-0000-0000-0000-000000000000"

2. Extraire les sections :
   get_many_select("Section",
     "Code, Name, OrderBy, Weight, Level, Constant, AllowCopying, IsTable, ParentSection.Code, Form.Code",
     where='Form.Code == "MON_FORM" && EntityState == "Active"',
     order_by="OrderBy", take=200)

3. Extraire les questions :
   get_many_select("FormQuestion",
     "Code, Reference, Order, Description, Question, DefaultMessage, QuestionType,
      Section.Code, ResponsePresetCategory.Code, Weight, IsRequired,
      ActivateComment, ActivateAttachment, ActivateWithResponseMultiple,
      ShowDescription, NotTakenInTheCalculation, DisplayInSelectList,
      ActivateQuestionValidation, RegEx, RegExValidationText, DecimalLength,
      DefaultValue, Formula, ConditionalDisplay, MaxResponseWeight,
      DisplayHelpFromSideMenu",
     where='Form.Code == "MON_FORM" && EntityState == "Active"',
     order_by="Order", take=500)

4. Extraire les reponses :
   get_many_select("FormAnswer",
     "Code, Name, Order, ResponseWeight, Question.Code,
      ResponsePreset.Code, ResponseListCategory.Code, MaxSize, Formula",
     where='Question.Form.Code == "MON_FORM"', take=1000)

5. Extraire les langues actives :
   get_many_select("Culture", "CultureCode, IsDefault",
     where='EntityState == "Active"', take=20)


PHASE 2 — TRANSFORMATION (en xlsx)

6. Generer les 3 fichiers xlsx a partir des donnees extraites :
   - Formater avec en-tetes orange/gris
   - Expander les champs multi-langue (une colonne par culture)
   - Conserver les codes existants

7. Le consultant modifie les fichiers :
   - Ajouter/modifier/supprimer des lignes
   - ATTENTION : les lignes existantes non modifiees seront re-importees
     → risque de doublon si les codes existent deja en base


PHASE 3 — RE-IMPORT (preparation)

8. Avant re-import, nettoyer les donnees :
   Option A — Import uniquement des NOUVEAUX elements :
     - Supprimer du fichier toutes les lignes dont le Code existe deja en base
     - Ne garder que les nouvelles lignes
   Option B — Re-creation complete :
     - Supprimer toutes les sections/questions/reponses existantes du template
     - Puis importer le fichier complet
     - ⚠ La suppression en cascade peut etre complexe

9. Valider avec la checklist pre-import (voir section precedente)

10. Importer dans l'ordre : Sections → Questions → Reponses
```

**Recommandation** : Privilegier l'Option A (ajout incremental) pour eviter les pertes de donnees. L'Option B necessite un backup prealable et une connaissance approfondie des dependances.

---

## Requetes utiles

```
# Vue complete d'un formulaire
get_many_select("Form", "Code, Name, Version, Prefix, Description",
  where='Code == "MON_FORM" && EntityState == "Active"')

# Arborescence sections
get_many_select("Section",
  "Code, Name, OrderBy, Weight, Level, ParentSection.Code, IsTable",
  where='Form.Code == "MON_FORM" && EntityState == "Active"',
  order_by="Level", take=100)

# Questions avec leur type et poids
get_many_select("FormQuestion",
  "Code, Question, Order, Weight, QuestionType, IsRequired, Formula, ConditionalDisplay, Section.Code",
  where='Form.Code == "MON_FORM" && EntityState == "Active"',
  order_by="Order", take=200)

# Reponses et leurs poids (⚠ pas de filtre EntityState sur FormAnswer)
get_many_select("FormAnswer",
  "Code, Name, Order, ResponseWeight, IsDefault, Color, Question.Code",
  where='Question.Form.Code == "MON_FORM"', take=500)

# Formules
get_many_select("FormFormula",
  "Code, Formula, LocalizedDescription, Section.Code",
  where='Form.Code == "MON_FORM" && EntityState == "Active"', take=50)

# Intervalles de formule
get_many_select("FormulaInterval",
  "LeftOperator, RightOperator, LeftValue, RightValue, OrderBy, FormulaClass.Code, FormFormula.Code",
  where='FormFormula.Form.Code == "MON_FORM" && EntityState == "Active"', take=50)

# Campagnes et leur statut (⚠ pas de champ Code — utiliser Name)
get_many_select("Campaign",
  "Name, Frequency, DeploymentStatus, DeploymentDate, StartCollectingDate, EndCollectingDate",
  where='EntityState == "Active"', take=20)

# Questionnaires d'une campagne
get_many_select("Questionary",
  "Code, Name, Version, QuestionsProgressionPercentage, IsEditable, Organization.Name",
  where='Campaign.Name == "MA_CAMPAGNE" && EntityState == "Active"', take=50)

# Reponses d'un questionnaire (⚠ pas de Code — utiliser Response, Question.Code)
get_many_select("UserResponse",
  "Question.Code, Response, DigitalNumber, OpenQuestionText, IsValidatedQuestion",
  where='Questionary.Code == "MON_QUESTIONNAIRE"', take=200)

# Conditions d'affichage (⚠ pas de Code — utiliser Name)
get_many_select("QuestionDisplayCondition",
  "Name, Question.Code",
  where='Question.Form.Code == "MON_FORM"', take=50)

# Workflow formulaire
get_many_select("FormWorkflow",
  "Form.Code, EntityWorkflowStatus.Code",
  where='Form.Code == "MON_FORM"', take=10)

# Versions d'un formulaire
get_many_select("Form", "Code, Name, Version, Description",
  where='Code == "MON_FORM"', order_by="Version DESC", take=10)
```
