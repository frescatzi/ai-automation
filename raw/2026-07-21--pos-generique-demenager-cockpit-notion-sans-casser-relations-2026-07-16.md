---
type: raw
title: "POS-GENERIQUE_demenager-cockpit-notion-sans-casser-relations_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_demenager-cockpit-notion-sans-casser-relations_2026-07-16.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Déménager un cockpit Notion (base + vues liées + file de validation) vers une sous-page sans rien casser

Date : 16.07.2026 · Portée : tout workspace Notion piloté par API (MCP Notion)
Principe : les bases et vues **déménagent**, elles ne se copient jamais. Les relations pointent vers les data sources, pas vers l'emplacement — un déménagement bien fait ne casse rien.

## Contexte d'usage

Une page « hub » a accumulé une base de tâches, des vues filtrées et des files de travail. Vous voulez restructurer : créer une sous-page dédiée (cockpit/dashboard) et y déplacer le tout, en gardant relations, rollups, filtres et tris intacts.

## Marche à suivre

1. **Cartographier avant de toucher** : fetch des pages source pour identifier chaque bloc à déplacer — la database elle-même (`<database inline>`) vs les blocs linked-view (wrappers « View of … ») ont des IDs distincts.

2. **Créer la page de destination** (create-pages, icône + intro).

3. **Déménager avec l'outil de move** (pas par édition de contenu) : `move-pages` accepte pages, databases ET blocs linked-view, en lot. L'ordre des appels = l'ordre d'empilement dans la destination.

4. **Recréer les titres de section** dans la destination (update_content, insertion au-dessus des blocs) — les headings/textes d'accompagnement ne suivent pas les blocs déplacés.

5. **Ne PAS réparer trop vite un affichage vide** : après un move, une linked view peut sembler vide dans le navigateur alors qu'elle est intacte. Diagnostic dans l'ordre : (a) fetch du bloc → la config (filtres/tris) est-elle là ? (b) query en mode view → les résultats reviennent-ils ? (c) si oui aux deux : simple cache → reload de la page navigateur.

6. **Réorganiser la page source** en `replace_content` : ⚠️ référencer TOUS les blocs enfants survivants (`<page>`, `<database>`) dans le nouveau contenu — un enfant non référencé est supprimé (ou l'appel refuse via le garde-fou).

7. **Vérifier en 3 points** : (a) re-fetch de la source → les blocs ont bien disparu (pas d'annulation silencieuse) ; (b) fetch d'un item de la base → ancestor-path = nouveau chemin + relations/rollups présents ; (c) query mode view d'une vue déplacée → résultats corrects.

8. **Wiki/knowledge adjacent** : si la restructuration inclut une nouvelle base de connaissances, `create-database` la crée en pleine page → `is_inline: true` pour l'afficher dans le corps ; les premières entrées **référencent** les documents existants (lien) au lieu de les copier.

## Difficultés rencontrées

- Vues linked-view affichées vides dans le navigateur juste après le move (panique « annulation silencieuse »).
- `replace_content` supprime les enfants non référencés.
- Base créée par API invisible dans la page (naît full-page).

## Solutions implémentées

- Diagnostic API avant réparation (fetch config + query view) → conclusion cache navigateur → reload suffit.
- Relisting systématique des enfants dans tout `replace_content`.
- `is_inline: true` post-création.

## Lessons learned

- **Utiliser l'outil de move, pas l'édition de contenu**, pour déplacer databases et linked views : plus atomique, gère les lots, préserve filtres/tris/permissions.
- **Les relations Notion sont attachées aux data sources** : déplacer pages et bases ne casse ni relations ni rollups.
- **Affichage ≠ état** : toujours vérifier l'état par API avant de conclure qu'un move a échoué ; le navigateur peut servir un cache obsolète.
- **Tout `replace_content` est une opération destructive potentielle** : inventorier les enfants d'abord.
- La vérification post-move en 3 points (source vidée / item re-parenté / vue interrogeable) prend une minute et attrape tous les modes d'échec connus.
