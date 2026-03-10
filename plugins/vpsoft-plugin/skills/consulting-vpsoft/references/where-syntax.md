# Syntaxe WHERE — System.Linq.Dynamic.Core

Reference complete pour construire des clauses WHERE valides dans les outils MCP VPSoft.

## Regle obligatoire

**Toujours inclure `EntityState == "Active"`** dans chaque requete WHERE pour exclure les donnees archivees/supprimees.

```
EntityState == "Active"
EntityState == "Active" && Name == "Test"
```

Valeurs de EntityState :
- `"Active"` (ou 1) — Enregistrement actif
- 0 — Draft (brouillon)
- 2 — Archived
- 3 — Deleted

## Operateurs de comparaison

| Operateur | Description | Exemple |
|-----------|-------------|---------|
| `==` | Egalite (JAMAIS `=`) | `Name == "Test"` |
| `!=` | Different | `Status != "Closed"` |
| `>` | Superieur | `Surface > 100` |
| `>=` | Superieur ou egal | `Surface >= 100` |
| `<` | Inferieur | `Surface < 500` |
| `<=` | Inferieur ou egal | `Surface <= 500` |

## Operateurs logiques

| Operateur | Description | Exemple |
|-----------|-------------|---------|
| `&&` ou `and` | ET logique | `A == "x" && B == "y"` |
| `\|\|` ou `or` | OU logique | `A == "x" \|\| A == "y"` |
| `!` | NON logique | `!(Type == "Archive")` |

## Types de valeurs

| Type | Format | Exemple |
|------|--------|---------|
| String | Double quotes | `Name == "Bureau Paris"` |
| Int / Decimal | Sans quotes | `Surface >= 100`, `Price == 45.67` |
| Boolean | Minuscule | `IsActive == true` |
| Null | Minuscule | `Description != null` |
| Date (ISO) | Double quotes | `CreatedDate >= "2024-01-01T00:00:00"` |
| GUID | Double quotes | `Id == "550e8400-e29b-41d4-a716-446655440000"` |

## Proprietes imbriquees (FK)

Navigation via notation point :

```
Organization.Name == "ACME"
Organization.Code == "FR001"
Building.Organization.Name == "ACME"
DynamicFolder.DynamicModule.Code == "HSE"
```

## Operateurs NON supportes

| Interdit | Alternative |
|----------|-------------|
| `=` (simple) | `==` |
| `Like` | Non disponible — faire du filtrage cote client |
| `In("a", "b")` | Utiliser OR : `(A == "a" \|\| A == "b")` |
| `Contains("test")` | Non fiable — preferer la recherche exacte |

## Erreurs courantes

| Erreur | Correction |
|--------|------------|
| `Name = "test"` | `Name == "test"` |
| `EntityState == Active` | `EntityState == "Active"` |
| `Surface == "100"` | `Surface == 100` (nombre sans quotes) |
| `IsActive == "true"` | `IsActive == true` (booleen sans quotes) |
| Oubli EntityState | Toujours ajouter `EntityState == "Active"` |
| Guillemets simples | Toujours utiliser les double quotes `"..."` |

## Exemples valides

```
# Filtrage de base
EntityState == "Active"

# Par nom
EntityState == "Active" && Name == "Bureau Paris"

# Numerique
EntityState == "Active" && Surface >= 100

# FK
EntityState == "Active" && Organization.Code == "FR001"

# Multi-conditions
EntityState == "Active" && Surface >= 50 && Surface <= 200

# OU
EntityState == "Active" && (Type == "Bureau" || Type == "Salle")

# Date
EntityState == "Active" && CreatedDate >= "2024-01-01T00:00:00"

# GUID
EntityState == "Active" && Organization.Id == "550e8400-e29b-41d4-a716-446655440000"

# NOT
EntityState == "Active" && !(Type == "Archive")

# Null check
EntityState == "Active" && Description != null

# Booleen
EntityState == "Active" && IsActive == true

# Recherche entite config
EntityName == "Ticket"   (pas de EntityState sur DynamicSettings)

# Recherche de code dynamique
EntityName == "Ticket" && PostCreateExpression != null
```

## Notes techniques

- Le parseur est base sur System.Linq.Dynamic.Core
- Les expressions sont converties en `Expression<Func<TEntity, bool>>`
- Les noms de proprietes sont **sensibles a la casse** (PropertyName exact)
- Toujours utiliser `get_entity_fields()` pour connaitre les PropertyName exacts
- Tester avec `get_entity_sample_data()` avant des requetes complexes
- Les parentheses sont essentielles pour grouper les conditions complexes
