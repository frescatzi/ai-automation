---
type: raw
title: "POS-GENERIQUE_creer-espace-agents-vues-filtrees-api_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_creer-espace-agents-vues-filtrees-api_2026-07-16.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Créer un espace « agents » en vues filtrées par API (linked views Notion, zéro copie)

Date : 16.07.2026 · Portée : tout workspace Notion piloté par API (MCP Notion)
Principe : un espace dédié (agents, équipe, client…) n'a pas besoin de sa propre base — **des linked views filtrées sur la base unique** suffisent. Une seule vérité, plusieurs fenêtres.

## Contexte d'usage

Une base de tâches porte une propriété discriminante (ex. select « Executor » = Human/Agent, ou « Team », « Client »…). Vous voulez une page-espace qui montre uniquement le travail concerné (board d'activité + file d'attente de validation), sans dupliquer les données.

## Marche à suivre

1. **Inventorier la page cible** (fetch) : contenu existant à conserver, sous-pages enfants (elles devront être référencées lors de la restructuration).

2. **Créer la vue « activité »** : `create-view` avec :
   - `parent_page_id` = la page-espace (c'est CE paramètre qui crée un bloc linked view sur la page — équivalent « /linked » de l'UI ; `database_id` ajouterait un onglet à la base au lieu d'un bloc) ;
   - `data_source_id` = la data source de la base unique ;
   - `type = board`, configure : `FILTER "Propriété" = "valeur"; GROUP BY "Status"; SHOW …`.

3. **Créer la vue « file d'attente »** : idem, `type = table`, avec **filtres composés** — plusieurs directives `FILTER` séparées par `;` forment un ET logique :
   ```
   FILTER "Propriété" = "valeur"; FILTER "Status" = "À valider"; SORT BY "Échéance" ASC; SHOW …
   ```

4. **Re-fetch la page** pour récupérer les IDs des nouveaux blocs (ajoutés en fin de page).

5. **Restructurer la page** (`replace_content`) : intro + sections titrées (## Activité / ## À valider / ## Base de connaissances…) avec les blocs vues aux bons endroits et le contenu existant recopié à l'identique. ⚠️ Référencer TOUS les enfants (`<page>`, `<database>`) sous peine de suppression.

6. **Cosmétique navigateur** (seul volet hors API) : masquer le titre de la base au-dessus de chaque linked view — icône sliders (View settings) → Layout → « Show data source title » OFF. La vue affiche alors son propre nom.

7. **Tester le flux bout-en-bout** : basculer une fiche test sur la valeur discriminante → query en mode view de chaque fenêtre concernée (l'espace dédié ET les vues « miroir » côté propriétaire) → la même fiche doit apparaître partout où le filtre matche. Remettre la fiche test dans son état initial après validation.

## Difficultés rencontrées

- Croyance héritée « linked views = navigateur uniquement ».
- Syntaxe du ET logique dans le DSL de configuration non évidente.
- Masquage du titre de source absent du DSL.
- Scroll d'automation navigateur capricieux sur les pages à boards horizontaux.

## Solutions implémentées

- `create-view` + `parent_page_id` : bloc linked view créé par API, filtres/tri/groupes/propriétés dans le même appel.
- `FILTER` répété = groupe AND (vérifié dans l'advancedFilter retourné).
- Toggle titre au navigateur, 4 clics par vue.
- `cmd+up` pour revenir en haut de page.

## Lessons learned

- **La séparation d'espaces est un problème de VUES, pas de bases** : tant qu'une propriété discriminante existe, tout espace dédié se construit en vues filtrées — réversible, sans migration, sans divergence.
- **L'API couvre désormais la création complète de linked views** ; ne garder le navigateur que pour les réglages cosmétiques non exposés.
- **Un « pont » entre espaces** (file de validation visible des deux côtés) = deux vues avec des filtres qui se recouvrent — aucune mécanique supplémentaire.
- Tester par requête (mode view) avant la démo visuelle : l'affichage navigateur peut retarder, la requête dit la vérité.
