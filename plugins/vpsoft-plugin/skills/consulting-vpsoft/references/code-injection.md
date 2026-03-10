# Points d'injection de code — VPSoft AGL

Reference complete des 38+ points d'injection de code disponibles dans la configuration VPSoft.

## Code C# sur DynamicSettings (12 points)

Champs de l'entite DynamicSettings contenant du code C# serveur :

| Champ | Mode | Description |
|-------|------|-------------|
| `UsingsExpression` | Global | Directives using C# |
| `InitializeCreateExpression` | Create | Initialisation du formulaire de creation |
| `InitializeEditExpression` | Edit | Initialisation du formulaire d'edition |
| `InitializeDetailsExpression` | Details | Initialisation de la fiche detail |
| `ValidationCreateExpression` | Create | Validation avant sauvegarde (creation) |
| `ValidationEditExpression` | Edit | Validation avant sauvegarde (edition) |
| `InjectionCreateExpression` | Create | Injection de code a la creation |
| `InjectionEditExpression` | Edit | Injection de code a l'edition |
| `PostCreateExpression` | Create | Code execute apres la creation |
| `PostEditExpression` | Edit | Code execute apres l'edition |
| `ListenerCreateExpression` | Create | Listener evenementiel (creation) |
| `ListenerEditExpression` | Edit | Listener evenementiel (edition) |

Requete pour lire tous les codes C# d'une entite :
```
get_many_select("DynamicSettings",
  "EntityName, UsingsExpression, InitializeCreateExpression, InitializeEditExpression,
   InitializeDetailsExpression, ValidationCreateExpression, ValidationEditExpression,
   InjectionCreateExpression, InjectionEditExpression, PostCreateExpression,
   PostEditExpression, ListenerCreateExpression, ListenerEditExpression",
  where='EntityName == "MonEntite"')
```

## Code frontend sur DynamicSettings (15 points)

| Champ | Type | Mode |
|-------|------|------|
| `ContentCssCreateExpression` | CSS | Create |
| `ContentCssEditExpression` | CSS | Edit |
| `ContentCssDetailsExpression` | CSS | Details |
| `ContentCssListExpression` | CSS | List |
| `ContentCssGlobalExpression` | CSS | Global |
| `ContentJavascriptCreateExpression` | JS | Create |
| `ContentJavascriptEditExpression` | JS | Edit |
| `ContentJavascriptDetailsExpression` | JS | Details |
| `ContentJavascriptListExpression` | JS | List |
| `ContentJavascriptGlobalExpression` | JS | Global |
| `ContentRazorCreateExpression` | Razor | Create |
| `ContentRazorEditExpression` | Razor | Edit |
| `ContentRazorDetailsExpression` | Razor | Details |
| `ContentRazorListExpression` | Razor | List |
| `ContentRazorGlobalExpression` | Razor | Global |

## Boutons Save (sur DynamicSettings)

| Champ | Description |
|-------|-------------|
| `SaveButtonJavascriptCallBack` | JS callback au clic Save (mode create) |
| `SaveButtonJavascriptCallBackForEdit` | JS callback au clic Save (mode edit) |
| `AfterSaveAction` | Action apres save (enum) |
| `ButtonSaveAndNew` | Afficher bouton Save & New (bool) |

## Code sur DynamicField (3 points)

| Champ | Mode | Description |
|-------|------|-------------|
| `DynamicPageFieldAdd` | Create | Code Razor injecte sur le champ en creation |
| `DynamicPageFieldEdit` | Edit | Code Razor injecte sur le champ en edition |
| `DynamicPageFieldDetails` | Details | Code Razor injecte sur le champ en detail |

```
get_many_select("DynamicField",
  "PropertyName, DynamicPageFieldAdd, DynamicPageFieldEdit, DynamicPageFieldDetails",
  where='EntityName == "MonEntite" && (DynamicPageFieldAdd != null || DynamicPageFieldEdit != null || DynamicPageFieldDetails != null)
   && EntityState == "Active"', take=50)
```

## Code sur DynamicButton (2 points)

| Champ | Description |
|-------|-------------|
| `DynamicExpression` | Code C# pour la visibilite du bouton |
| `JSCallBack` | Code JavaScript execute au clic |

```
get_many_select("DynamicButton",
  "Code, LocalizedName, Icon, DynamicButtonPosition, DynamicExpression, JSCallBack",
  where='DynamicSettings.EntityName == "MonEntite" && EntityState == "Active"', take=20)
```

## Entites de code autonomes

| Entite | Champs code | Description |
|--------|-------------|-------------|
| `DynamicFunction` | CodeUsing, CodeFunction, CodeClass | Fonctions C# reutilisables |
| `DynamicClass` | CodeUsing, CodeClass | Classes C# partagees |
| `DynamicBatch` | CodeBatch, ClassBatch, CodeReferenceBatch | Traitements planifies |
| `DynamicConst` | CodeCSharp, Value | Constantes (Text, Session, Cache) |
| `DynamicPage` | CodeRazor, CodeCss, CodeJavascript | Pages custom (front) |
| `DynamicWidget` | CodeRazor, CodeCss, CodeJavascript | Widgets dashboard |
| `DynamicGlobalCode` | CodeRazor, CodeCss, CodeJavascript | Code global app/module |

## Code sur EntityWorkflow

| Champ | Entite | Description |
|-------|--------|-------------|
| `Action` | EntityWorkflowAction | Code C# execute a la transition |
| `Script` | EntityWorkflowSetByScript | Code C# pour calcul de valeur |

```
get_many_select("EntityWorkflowAction",
  "Code, LocalizedName, Action",
  where='EntityWorkflow.EntityName == "MonEntite" && Action != null && EntityState == "Active"', take=20)
```

## Rechercher du code contenant un terme

Pour trouver ou un terme est utilise dans le code :
```
# Dans les DynamicSettings (12 champs C#)
get_many_select("DynamicSettings",
  "EntityName, PostCreateExpression",
  where='PostCreateExpression != null', take=50)
# Puis filtrer cote client pour le terme recherche

# Dans les DynamicFunction
get_many_select("DynamicFunction",
  "Name, CodeFunction",
  where='EntityState == "Active"', take=100)
# Puis filtrer cote client

# Dans les DynamicBatch
get_many_select("DynamicBatch",
  "Name, CodeBatch",
  where='EntityState == "Active"', take=100)
```

Note : Le WHERE ne supporte pas `Contains()` de maniere fiable. Recuperer les donnees et filtrer cote client.
