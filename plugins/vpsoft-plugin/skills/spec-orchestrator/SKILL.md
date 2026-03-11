---
name: spec-orchestrator
description: "Orchestrateur de specification fonctionnelle LocaParc. Lance des agents autonomes en parallele pour documenter les vues listes et formulaires VPSoft de chaque entite. Declencheur : specifier les vues listes et formulaires du document de specs, completer les chapitres du docx de specification, generer les specs depuis VPSoft. Resout le probleme de depassement de contexte en isolant chaque entite dans un agent separe."
---

# Spec Orchestrator — Agents paralleles pour specifications LocaParc

Skill d'orchestration qui lance des **agents autonomes** (outil Agent, subagent_type: "general-purpose") pour documenter chaque entite VPSoft en parallele, evitant ainsi le depassement de la fenetre de contexte.

**Reference technique :** Lire `references/agent-spec-entity.md` pour le prompt complet des agents.

---

## Principe

```
Orchestrateur (cette skill)
  |
  |-- Etape 1 : Lire le docx source → identifier les sous-sections vides
  |-- Etape 2 : Resoudre chaque sous-section → EntityName VPSoft
  |-- Etape 3 : Lancer N agents en parallele (1 par entite)
  |-- Etape 4 : Assembler les resultats dans le format du docx
```

Chaque agent a son propre contexte isole et fait ses propres appels MCP.

---

## Etape 1 — Analyser le document source

Extraire la structure du docx :
```bash
pandoc "{docx_path}" -o /tmp/spec_structure.md
```

Identifier les chapitres contenant des sous-sections avec "Vue liste" et "Formulaire" vides.

Structure attendue du docx LocaParc (les titres peuvent varier) :

| Chapitre | Sous-sections |
|----------|--------------|
| Locations | Bailleurs externes, Locations 1er bail, Locataires, Locations 2nd bail |
| Demandes | Demandes de logement, Demandes d'intervention, Demandes de depart |
| Logements | Biens, Annonces logements, Recherche de logement |
| Evenements | Visites, EDLE, EDLS |
| Signatures | Demandes de signature, Signataires |
| Gestion des contrats | Contrats de location, Contrats d'assurance, Charges |
| Facturation | Loyers, Factures ponctuelles |

---

## Etape 2 — Resoudre les entites VPSoft

Faire UN appel MCP initial pour recuperer toutes les entites :
```
get_all_entities()  # MCP LocaParc (0839c98b)
```

Puis mapper chaque sous-section du docx vers son EntityName.
Si le mapping est ambigu, utiliser get_entity_fields sur les candidats pour confirmer.

---

## Etape 3 — Lancer les agents

Lire le fichier `references/agent-spec-entity.md` pour obtenir le prompt template des agents.

Pour chaque entite identifiee, construire le prompt de l'agent en remplacant `{entity_name}` et `{section_title}` dans le template.

**Lancer les agents par lots de 3-4 maximum** pour eviter la surcharge.
Utiliser l'outil Agent avec `subagent_type: "general-purpose"`.

### Template de prompt agent

```
Tu documentes l'entite VPSoft "{entity_name}" (titre metier : "{section_title}") pour les specifications fonctionnelles LocaParc.

[Coller ici le contenu de references/agent-spec-entity.md]

IMPORTANT : Utilise les outils MCP dont le nom contient "0839c98b" (instance LocaParc).
L'entite a documenter est : {entity_name}
Le titre de la section est : {section_title}

Produis UNIQUEMENT le bloc markdown de sortie (vue liste + formulaire par onglet).
Pas d'introduction, pas de conclusion, juste le contenu structuré.
```

### Lancement parallele

```
# Lot 1 — Chapitre "Locations"
Agent("spec Bailleurs externes", prompt=prompt_bailleur, subagent_type="general-purpose")
Agent("spec Locations 1er bail", prompt=prompt_loc1, subagent_type="general-purpose")
Agent("spec Locataires", prompt=prompt_locataire, subagent_type="general-purpose")
Agent("spec Locations 2nd bail", prompt=prompt_loc2, subagent_type="general-purpose")

# Attendre les resultats du lot 1, puis lot 2, etc.
```

---

## Etape 4 — Assembler les resultats

Chaque agent renvoie un bloc markdown autonome. L'orchestrateur :

1. Collecte tous les blocs
2. Les ordonne selon la structure du docx source
3. Les formate au format attendu (celui de l'exemple "Bailleurs externes")
4. Produit soit :
   - Un fichier markdown compilé (rapide, pour review)
   - Un docx final via la skill docx (pour integration directe)

---

## Gestion des erreurs

- Si un agent echoue (entite introuvable, MCP timeout), noter l'echec et continuer avec les autres
- Produire un rapport d'erreurs en fin d'execution
- Les entites en echec peuvent etre relancees individuellement

---

## Prompt d'utilisation recommande

### Un chapitre a la fois (recommande)
```
En utilisant la skill spec-orchestrator, complete les specifications
du chapitre "Locations" du fichier
04_SPECIFICATIONS/Locaparc_specifications_fonctionnelles_applicatives - v1.0.docx

Lance un agent par sous-section (Bailleurs externes, Locations 1er bail,
Locataires, Locations 2nd bail) pour documenter les vues listes et formulaires
depuis la configuration VPSoft LocaParc.
```

### Tous les chapitres (attention au temps)
```
En utilisant la skill spec-orchestrator, complete TOUS les chapitres
de specifications du fichier docx.
Lance les agents par lots de 3-4 pour chaque chapitre.
Assemble les resultats dans un fichier markdown compile.
```

### Une seule entite (debug/test)
```
En utilisant la skill spec-orchestrator, documente uniquement l'entite
"Bailleurs externes" avec un agent. C'est un test avant de lancer
les autres.
```