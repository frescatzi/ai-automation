---
type: raw
title: "POS-EXACT_Backstage-v3_N3-agent-space_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_n3-agent-space_2026-07-16.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · N3 « Agent Space » (le monde agent)

Date : 16.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **N3 FAIT & VALIDÉ par Karter le 16.07.2026** (tâche test remise en 🧑 Human après validation)
Référence : `SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md` (§4 N3) · `CONCEPT_Backstage-v3_Deux-Mondes_2026-07-16.md` (§2 amendé, §4)

## Objectif

Remodeler la page Agents (renommée « Agents Space » par Karter) en monde agent complet : **📋 Activity** (vue Tasks filtrée Executor=Agent, board par statut), **⏳ To validate** (Executor=Agent & Status=To Validate), **📚 Knowledge base** (contenu existant conservé). Zéro copie — uniquement des vues filtrées sur la base Tasks unique.

## Marche à suivre exacte

1. **Lire avant de modifier** : `notion-fetch` de la page Agents `39124f2fef0c81a99ba6e84df86498ff` — inventaire du contenu existant (intro, roster 8 agents en table, outils utilitaires, Hermès, sous-pages Brand Source of Truth `39124f2fef0c8152b145de5990e2945e` + Playbook `39124f2fef0c819dbf23df502e033d3f`).

2. **Créer la vue Activity** : `notion-create-view` avec `parent_page_id = 39124f2fef0c81a99ba6e84df86498ff` (⚠️ pas `database_id` — c'est ce paramètre qui crée un bloc *linked view* sur la page, équivalent de la commande UI « /linked »), `data_source_id = e50f931d-3f8d-483f-895a-c09408455128` (Tasks), `type = board`, configure :
   ```
   FILTER "Executor" = "🤖 Agent"; GROUP BY "Status"; SHOW "Task", "Executor", "Due Date", "Event", "Owner"
   ```
   → vue `view://39f24f2f-ef0c-81d4-86fe-000ce67edf14`, bloc ajouté en fin de page.

3. **Créer la vue To validate** : idem, `type = table`, configure avec **filtres composés** (2 directives FILTER = ET logique, vérifié dans l'advancedFilter retourné) :
   ```
   FILTER "Executor" = "🤖 Agent"; FILTER "Status" = "To Validate"; SORT BY "Due Date" ASC; SHOW "Task", "Executor", "Due Date", "Event", "Owner", "Notes"
   ```
   → vue `view://39f24f2f-ef0c-81f2-8914-000c22ff0461`.

4. **Récupérer les IDs des 2 nouveaux blocs** : re-fetch de la page → blocs `39f24f2fef0c8126a0d5dfeaa271c601` (Activity) et `39f24f2fef0c81d4bc69ed6c55615848` (To validate), ajoutés en fin de page.

5. **Restructurer la page** : `notion-update-page replace_content` — intro « The agent world… » + `## 📋 Activity` (+ desc) + bloc Activity + `## ⏳ To validate` (+ desc) + bloc To validate + `## 📚 Knowledge base` + tout le contenu existant recopié À L'IDENTIQUE (roster, outils, Hermès) + les 2 sous-pages référencées. ⚠️ Relister TOUS les enfants (`<page>`, `<database>`) sinon suppression.

6. **Masquer les titres de base** (navigateur, Claude in Chrome — seul volet non pilotable par API) : pour chaque vue → icône sliders (View settings) → **Layout** → toggle **Show data source title** OFF. Fait pour Activity et To validate ; les vues affichent alors leur propre nom (📋 Activity / ⏳ To validate) au lieu de « Tasks ».

7. **Test du pont (bout-en-bout)** : `update_properties` `{"Executor": "🤖 Agent"}` sur la tâche test `39e24f2fef0c8171ab51d607b6af539c` (« Approve aftermovie », déjà Status=To Validate) puis query mode **view** des 3 fenêtres :
   - vue Activity → 1 résultat ✅
   - vue To validate (Agent Space) → 1 résultat ✅
   - vue ✅ Awaiting validation du cockpit (accueil, `39f24f2fef0c81209f84e79ca8c809b3?v=39f24f2fef0c8143aa54000c238a5ada`) → le MÊME résultat ✅
   → le pont de validation fonctionne : une tâche agent en To Validate est visible des deux côtés, sans copie.

8. **Validation Karter** (16.07) → tâche test remise en 🧑 Human → POS + coche + maj mémoire.

## Difficultés rencontrées

- Croyance héritée : « les linked views se créent au navigateur uniquement » (pattern v2).
- Syntaxe DSL des filtres composés non documentée dans la description courte de l'outil (ET logique ?).
- « Show data source title » OFF absent du DSL de create-view/update-view → reste navigateur.
- Scroll de page erratique dans Notion via automation (la molette reste bloquée sur les boards à scroll horizontal).

## Solutions implémentées

- `notion-create-view` avec `parent_page_id` crée bien un bloc linked view sur la page (testé, immédiat).
- Deux directives `FILTER` séparées par `;` = groupe AND (vérifié dans l'advancedFilter JSON retourné par l'outil).
- Toggle fait au navigateur pour les 2 vues (sliders → Layout → Show data source title OFF).
- Raccourci `cmd+up` pour remonter en haut de page quand la molette ne répond plus.

## Lessons learned

- **Les linked views filtrées se créent par API** (`create-view` + `parent_page_id` + `data_source_id`) avec filtres, tri, group by et propriétés affichées dans le même appel — le navigateur n'est plus nécessaire que pour le cosmétique (masquer le titre de la source).
- **`FILTER` × n = ET logique** dans le DSL — suffisant pour « Executor=Agent AND Status=To Validate ».
- **Le pont de validation ne coûte rien** : aucune mécanique nouvelle, juste deux vues filtrées de plus sur la même base — la preuve du bien-fondé du contrat Executor (N1).
- Toujours **re-fetch la page après create-view** pour obtenir les IDs des blocs (nécessaires au replace_content de mise en ordre).
- Vérifier un pont/flux par **query mode view des deux côtés** avant la démo visuelle : rapide et sans ambiguïté.
