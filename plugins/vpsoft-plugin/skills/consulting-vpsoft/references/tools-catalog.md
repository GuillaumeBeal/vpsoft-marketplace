# Catalogue des outils MCP VPSoft

Reference complete des 15 outils MCP avec signatures, parametres et exemples.

---

## Exploration des donnees (API V2 — toutes les 233 entites)

### `invoke_vpsoft_function`
Invoque une fonction VPSoft custom (DynamicFunction) cote serveur.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `function_name` | str | oui | - | Nom de la fonction a invoquer |
| `parameters` | List[Dict] | non | [] | Parametres de la fonction |

```
invoke_vpsoft_function("MaFonctionCalcul", [{"Name": "param1", "Value": "valeur1"}])
```

---

### `get_entities_count`
Compte les entites correspondant a des criteres.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `entity_name` | str | oui | - | Nom de l'entite |
| `where` | str | non | "" | Clause WHERE |
| `filter_query` | str | non | "All" | Filtre additionnel |

```
get_entities_count("Building", where='EntityState == "Active" && Surface >= 100')
```

---

### `get_many_select`
Outil principal. Recupere des enregistrements avec selection de champs et filtrage.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `entity_name` | str | oui | - | Nom de l'entite |
| `select` | str | oui | - | Champs a recuperer (virgules) |
| `where` | str | non | None | Clause WHERE |
| `order_by` | str | non | None | Champ de tri (ex: "Name DESC") |
| `skip` | int | non | 0 | Offset pagination |
| `take` | int | non | 0 | Nombre de resultats (0 = tous) |
| `filter_query` | str | non | "All" | Filtre additionnel |

Supporte la notation point pour les FK : `Organization.Name, Building.Organization.Code`

```
get_many_select("Building", "Name, Surface, Organization.Name",
  where='EntityState == "Active" && Surface >= 100', order_by="Surface DESC", take=20)
```

---

### `get_all_entities`
Liste toutes les entites/tables disponibles (233+).

| Parametre | Aucun |
|-----------|-------|

Retourne EntityName et LocalizedEntityName pour chaque entite.
Implementation interne : requete DynamicSettings avec filtre ViewName == null.

```
get_all_entities()
```

---

### `get_entity_fields`
Recupere la structure complete des champs d'une entite.

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `entity_name` | str | oui | Nom technique de l'entite |

Retourne : PropertyName, PropertyType, Required, IsEntityCode, labels localises.
Implementation : 2 requetes (DynamicField + DynamicFieldName) fusionnees.

```
get_entity_fields("Ticket")
```

---

### `get_entity_sample_data`
Recupere un echantillon de donnees pour comprendre la structure.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `entity_name` | str | oui | - | Nom de l'entite |
| `select` | str | oui | - | Champs a recuperer |
| `limit` | int | non | 5 | Nombre d'enregistrements |

WHERE hardcode a `EntityState == "Active"`.

```
get_entity_sample_data("Ticket", "Code, Name, Organization.Name, Status", limit=10)
```

---

## CRUD (API v1 REST — 3 entites seulement)

**Limitation critique** : Seules Organization, Ticket, HSEIncident ont le REST v1 active.

### `create_entity`
Cree une entite via POST api/v1/Post/{entity_name}.

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `entity_name` | str | oui | Nom de l'entite |
| `entity_data` | dict | oui | Donnees (cle-valeur) |

Convention FK : suffixe `_Code_` avec la valeur IsEntityCode de l'entite liee.

```
create_entity("Ticket", {
  "Name": "Mon ticket",
  "Organization_Code_": "BOULOGNE",
  "MaintenanceEquipment_Code_": "Extincteur01"
})
```

---

### `update_entity_patch`
PATCH api/v1/Patch/{entity_name}/{entity_id}.

| Parametre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `entity_name` | str | oui | Nom de l'entite |
| `entity_id` | str | oui | GUID de l'entite |
| `entity_data` | dict | oui | Donnees partielles |

**⚠ NON FONCTIONNEL** — Retourne erreur 400 sur toutes les entites. Les endpoints Patch/Put sont concus pour l'import fichier, pas le JSON.

---

### `delete_entity`
DELETE api/v1/Delete/{entity_name}/{entity_id}.

Non expose directement comme outil MCP mais disponible via l'API.

---

### `get_entity_url`
Recupere l'URL d'une entite dans l'interface web.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `entity_name` | str | oui | - | Nom de l'entite |
| `url_action` | str | non | "Index" | Action : Index, Create, Edit, Details |
| `entity_id` | str | non | None | GUID de l'entite |
| `view_name` | str | non | None | Nom de la vue |

```
get_entity_url("Ticket", entity_id="GUID-ICI", url_action="Edit")
```

---

## Logs et Audit (V1)

### `get_app_errors`
Erreurs applicatives depuis AppLog.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `date_from` | str | non | None | Date debut ISO |
| `date_to` | str | non | None | Date fin ISO |
| `error_type` | str | non | None | "0" (ManagedLog/gerees) ou "1" (NotManagedLog/crashes) |
| `search` | str | non | None | Texte libre dans Message/StackTrace |
| `skip` | int | non | 0 | Offset |
| `take` | int | non | 20 | Nombre de resultats |

```
get_app_errors(search="NullReferenceException", take=10)
```

---

### `get_audit_history`
Modifications champ par champ depuis AuditData.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `entity_name` | str | non | None | Entite modifiee |
| `entity_id` | str | non | None | GUID specifique |
| `field_name` | str | non | None | Champ modifie |
| `action` | str | non | None | "0" (Insert), "1" (Update), "2" (Delete) |
| `user_name` | str | non | None | Nom utilisateur |
| `date_from` | str | non | None | Date debut ISO |
| `date_to` | str | non | None | Date fin ISO |
| `skip` | int | non | 0 | Offset |
| `take` | int | non | 20 | Nombre de resultats |

```
get_audit_history(entity_name="Building", entity_id="GUID", take=50)
```

---

### `get_audit_trail`
Journal d'acces utilisateur depuis AuditTrail.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `user_name` | str | non | None | Nom utilisateur |
| `area` | str | non | None | Zone d'acces |
| `ip_address` | str | non | None | Adresse IP |
| `date_from` | str | non | None | Date debut ISO |
| `date_to` | str | non | None | Date fin ISO |
| `skip` | int | non | 0 | Offset |
| `take` | int | non | 20 | Nombre de resultats |

```
get_audit_trail(user_name="Dupont", take=50)
```

---

### `get_mail_logs`
Emails envoyes depuis MailLog.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `to_address` | str | non | None | Destinataire (match exact) |
| `subject_search` | str | non | None | Recherche dans le sujet |
| `date_from` | str | non | None | Date debut ISO |
| `date_to` | str | non | None | Date fin ISO |
| `skip` | int | non | 0 | Offset |
| `take` | int | non | 20 | Nombre de resultats |

```
get_mail_logs(to_address="user@example.com", date_from="2024-06-01T00:00:00")
```

---

### `get_audit_versions`
Versions d'entites depuis AuditVersion.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `entity_name` | str | non | None | Entite |
| `entity_id` | str | non | None | GUID |
| `date_from` | str | non | None | Date debut ISO |
| `date_to` | str | non | None | Date fin ISO |
| `skip` | int | non | 0 | Offset |
| `take` | int | non | 20 | Nombre de resultats |

```
get_audit_versions(entity_name="Building", entity_id="GUID")
```

---

### `get_cascade_logs`
Operations cascade depuis CascadeEntityProcessLog.

| Parametre | Type | Requis | Default | Description |
|-----------|------|--------|---------|-------------|
| `table_name` | str | non | None | Table cible |
| `status` | str | non | None | "0"-"5" (Pending/Starting/Calculating/Processing/Success/Error) |
| `operation_type` | str | non | None | "0" (Delete), "1" (Update), "2" (Retrieve) |
| `date_from` | str | non | None | Date debut ISO |
| `date_to` | str | non | None | Date fin ISO |
| `skip` | int | non | 0 | Offset |
| `take` | int | non | 20 | Nombre de resultats |

```
get_cascade_logs(status="5")  # Erreurs uniquement
```

---

## Resources et Prompts

### Resource `vpsoft://app_info`
Informations sur l'application VPSoft (base URL, outils disponibles, contexte metier).

### Prompt `get_vpsoft_api_guide`
Guide complet de l'API VPSoft (structure, syntaxe WHERE, conventions, exemples).
