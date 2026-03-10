# Claude

Ce fichier rassemble des notes, modèles de prompts et exemples pour utiliser Claude (assistant IA) dans les projets Consulting VPSoft.

## Objectif

Fournir des templates de prompts, bonnes pratiques et exemples concrets pour obtenir des réponses utiles et sûres de Claude.

## Prompts modèles (templates)

- **Résumé concis :**
  > Résume en 3 phrases le texte suivant en mettant en avant les actions, résultats et risques : "<TEXTE>".

- **Génération de code C# :**
  > Écris une méthode C# nommée `<Nom>` qui prend `<paramètres>` et retourne `<type>`. Utilise `<librairie>` si besoin. Inclue un exemple d'appel et un test unitaire xUnit.

- **Refactorisation :**
  > Refactore le code suivant pour améliorer lisibilité et performance sans changer le comportement :
  > ```csharp
  > <CODE>
  > ```

- **Débogage / Analyse d'erreur :**
  > J'obtiens l'exception suivante : "<STACKTRACE>". Voici le code : `<CODE>`. Explique la cause probable et propose une correction précise.

- **Extraction de spécifications :**
  > À partir du texte suivant, liste les exigences fonctionnelles numérotées, les contraintes techniques et les critères d'acceptation : "<TEXTE>".

- **Génération de tests unitaires :**
  > Génère des tests unitaires xUnit pour la méthode suivante, en couvrant cas nominal, cas limites et erreurs attendues : `<CODE>`.

- **Revue sécurité rapide :**
  > Analyse ce fragment pour vulnérabilités (injections, XSS, fuite de données). Propose des mitigations concrètes : `<CODE>`.

- **Rédaction de documentation technique :**
  > Rédige une fiche technique (200–400 mots) expliquant le composant `<NOM>` : but, API publique, exemples d'usage et limitations.

- **Traduction technique :**
  > Traduis le texte suivant en français technique, en conservant la terminologie métier : "<TEXTE>".

## Exemples détaillés

- **Exemple — Génération C# (prompt rempli) :**
  Prompt :
  > Écris une méthode C# nommée `CalculateTotalPrice` qui prend une liste d'items `(decimal price, int qty)`, applique un taux de TVA de 20% et retourne le total. Fournis un exemple d'appel et un test xUnit.

  Réponse attendue (structure) :
  - Méthode C# complète avec signature, validation des paramètres et calcul.
  - Exemple d'appel dans un petit snippet `Main`.
  - Test xUnit couvrant cas normal et quantité nulle.

- **Exemple — Extraction de specs (prompt rempli) :**
  Prompt :
  > À partir du texte suivant, extrais les exigences fonctionnelles et critères d'acceptation : "L'utilisateur doit pouvoir importer un fichier CSV de commandes, valider le format, et recevoir un rapport d'erreurs." 

  Réponse attendue (structure) :
  - RF1 : Import CSV — description, format attendu.
  - RF2 : Validation — règles et messages d'erreur.
  - Critères d'acceptation : cas de test minimal.

## Bonnes pratiques

- Masquer ou anonymiser les données sensibles avant envoi.
- Fournir toujours un contexte minimal mais suffisant (exemples d'entrée attendue).
- Demander des retours structurés : "Donne la réponse en 3 sections : résumé, modifications proposées, code corrigé".

## Contact

Pour questions ou contributions, contacter l'équipe Consulting VPSoft.
