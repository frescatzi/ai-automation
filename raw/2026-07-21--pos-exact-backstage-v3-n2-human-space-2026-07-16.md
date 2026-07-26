---
type: raw
title: "POS-EXACT_Backstage-v3_N2-human-space_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_n2-human-space_2026-07-16.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · N2 « Human Space » (le monde humain)

Date : 16.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **N2 FAIT & VALIDÉ par Karter le 16.07.2026**
Référence : `SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md` (§4 N2) · `CONCEPT_Backstage-v3_Deux-Mondes_2026-07-16.md` (§2, §5)

## Objectif

Faire de Human Space un monde complet : 🔭 Cockpit (base Tasks + vues + file de validation), ⚙️ Operations, 📚 Knowledge (wiki humain NOUVEAU), 👥 Team. Tout déménage, rien n'est copié ; les relations des 10 bases restent intactes.

## Marche à suivre exacte

1. **Lire avant de modifier** : `notion-fetch` de Human Space `39124f2fef0c811599edec1c18b44b20` et The Backstage `39124f2fef0c8172a961c8a57721ff0f` pour cartographier les blocs à déménager (base Tasks + 3 linked views sur le Backstage ; file « ✅ Awaiting validation » sur Human Space).

2. **Créer 🔭 Cockpit** : `notion-create-pages` parent Human Space → page `39f24f2f-ef0c-8121-aa2c-d771f1414851` (icône 🔭, intro italique).

3. **Déménager la base Tasks** : `notion-move-pages` avec l'ID de la DATABASE `cd210f519f9a4d67a50ef6f50e851dc5` → new_parent = page Cockpit. ✅ Le bloc inline suit la database — pas besoin du vieux pattern « 2 contentUpdates dans le même appel ».

4. **Déménager les 3 vues cockpit** (blocs linked view = « databases » wrapper) : `notion-move-pages` avec les 3 IDs `39e24f2fef0c817ab3b5f50fee65eb1f` (⏭️ Up Next) · `39e24f2fef0c81d9b9bde18dff28b75c` (🎪 Upcoming Editions) · `39e24f2fef0c81519288e8db47d61f53` (📣 Publishing Pipeline) → Cockpit. Même appel, même parent.

5. **Déménager la file de validation** : `notion-move-pages` sur `39f24f2fef0c81209f84e79ca8c809b3` (linked view To Validate, depuis Human Space) → Cockpit, puis `notion-update-page update_content` sur Cockpit pour insérer le titre `## ✅ Awaiting validation` + description au-dessus du bloc.

6. **Déménager ⚙️ Operations** : `notion-move-pages` sur la page `39124f2fef0c81efa298e09b37626bb8` → Human Space.

7. **Créer 📚 Knowledge** : `notion-create-pages` parent Human Space → page `39f24f2f-ef0c-814e-ae67-e7a829f3c8c2` ; `notion-create-database` parent = cette page, schema `CREATE TABLE ("Name" TITLE, "Category" SELECT('SOP':green, 'Brand':pink, 'How-to':blue, 'Decisions':orange, 'Reference':gray), "Summary" RICH_TEXT, "Added" CREATED_TIME)` → db `b118536be07242e1beee2373393fed6b` / ds `7f499c43-0dc2-4ad1-b23b-6412363ff4e2` ; la base naît full-page → `notion-update-data-source` `is_inline: true` pour l'afficher dans la page.

8. **Première entrée du wiki** : `notion-create-pages` parent = ds Knowledge → « 📘 Budget How-To » (`39f24f2f-ef0c-81a8-9a1e-e1e6225f6a40`), Category=SOP, contenu = lien vers le SOP existant `39f24f2fef0c81fab937d4b2fe1bdb53` (référence, PAS de copie).

9. **Réorganiser Human Space** : `notion-update-page replace_content` — intro + `## 🚪 Rooms` + les 4 accès dans l'ordre Cockpit / Operations / Knowledge / Team. ⚠️ Référencer TOUS les blocs enfants (`<page>` / `<database>`) dans le nouveau contenu, sinon l'appel échoue ou supprime des enfants (garde-fou `allow_deleting_content`).

10. **Vérifications** :
    - Re-fetch The Backstage → base Tasks et 3 vues absentes (move non annulé).
    - Fetch d'une tâche exemple → ancestor-path `Tasks → Cockpit → Human Space → The Backstage`, relations Event/Venue et rollups Budget présents.
    - `notion-query-data-sources` mode **view** sur Up Next → les 4 tâches reviennent (filtres/tris intacts après move).
    - Navigateur : reload de la page Cockpit → vues peuplées (Up Next 4 tâches + alertes Overdue, Upcoming Editions EP6).

11. **Validation Karter** (16.07) → POS + coche + maj mémoire.

## Difficultés rencontrées

- Juste après le move, les linked views paraissaient **vides dans le navigateur** (état « Edit filters / New page ») alors que la config API était intacte — grosse frayeur type « annulation silencieuse ».
- `replace_content` sur Human Space risquait de supprimer les pages/bases enfants non référencées.
- La base Knowledge créée par `create-database` naît en pleine page, invisible dans le corps de la page parente.

## Solutions implémentées

- Diagnostic en 2 temps : fetch du bloc vue (config intacte) + query en mode view (4 résultats) → conclusion : simple cache d'affichage ; un **reload de la page** dans le navigateur a tout affiché.
- Nouveau contenu de Human Space rédigé en référençant explicitement les 4 enfants (3 pages + database Team).
- `is_inline: true` via update-data-source juste après création.

## Lessons learned

- **`notion-move-pages` déménage proprement pages, databases ET blocs linked-view** — plus fiable et plus simple que l'ancien pattern « 2 contentUpdates même appel ». On garde quand même le réflexe re-vérifier après ~1 min.
- **Une vue qui semble vide après un move n'est pas forcément cassée** : vérifier la config par API + query mode view AVANT de « réparer » ; souvent c'est le cache du navigateur → reload.
- **Les relations/rollups survivent aux déménagements** (confirmé sur Event/Venue/Budget) : les relations pointent vers les data sources, pas vers l'emplacement des pages.
- **`replace_content` = danger enfants** : toujours relister les blocs `<page>`/`<database>` existants dans le nouveau contenu.
- **`create-database` naît full-page** : penser `is_inline: true` si la base doit vivre dans le corps de la page.
- Un wiki qui référence (lien vers le SOP) plutôt que copie = zéro divergence de contenu — cohérent avec « une seule vérité ».
