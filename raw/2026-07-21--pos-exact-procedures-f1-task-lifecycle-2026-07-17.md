---
type: raw
title: "POS-EXACT_Procedures_F1-task-lifecycle_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_procedures_f1-task-lifecycle_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Fiche procédure « Task Lifecycle » (wiki 📚 Knowledge, AFTRSN Backstage)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Type : POS-EXACT (marche à suivre réelle, avec IDs et appels API).
Livrable documenté : première fiche de procédure du wiki humain, catégorie SOP.

## Contexte
Session « création des documents de procédures ». Cadrage validé par Karter : lot 1 = trio humain (Task lifecycle, Validate an agent's work, The Cockpit), format MVP, humains d'abord. Cette fiche = SOP « Task Lifecycle ».

## Cible Notion
Wiki 📚 Knowledge : page `39f24f2fef0c814eae67e7a829f3c8c2` · db `b118536be07242e1beee2373393fed6b` · ds `7f499c43-0dc2-4ad1-b23b-6412363ff4e2` (propriétés Name, Category [SOP/Brand/How-to/Decisions/Reference], Summary).

## Étapes réalisées
1. Vérif connecteur Notion (obligatoire en début de session) : `notion-fetch` sur la page wiki → OK.
2. Lecture du schéma cible pour décrire le système RÉEL :
   - `notion-fetch` ds Tasks `e50f931d-3f8d-483f-895a-c09408455128` : statuts (To Do, In Progress, To Validate, Blocked, Done), Executor (🧑 Human / 🤖 Agent au moment T), Owner (Katel/Jack/Jacques/Sara), Category, Event, Due Date, template New Task `39e24f2fef0c802695c8ccb3accb22e0`.
   - `notion-fetch` ds Knowledge (Name/Category/Summary) + entrée modèle « 📘 Budget How-To » `39f24f2f-ef0c-81a8-9a1e-e1e6225f6a40` → principe confirmé : le corps RÉFÉRENCE par lien, ne copie pas.
3. Création de la fiche par API : `notion-create-pages`, parent = `{ data_source_id: 7f499c43-... }`. Propriétés : Name « 🔄 Task Lifecycle », Category SOP, Summary. Contenu Markdown au format MVP (When to use / Where tasks live / Create a task [6 étapes] / Move it through its life [5 statuts] / Common mistakes / Owner / Related). Page créée : `3a024f2f-ef0c-81e6-8d24-db5a79a8630c`.
4. Corrections demandées, par `notion-update-page` (command `update_content`, search-and-replace ciblé) :
   a. Owner : « Founders (Karter) » corrigé en « Founder (Katel) ». Deux tentatives en échec car Karter éditait la page en direct au navigateur (le texte stocké différait) ; re-fetch puis match sur la valeur réelle « Founder (Kartel) » → « Founder (Katel) ».
   b. Renommage structurel en cours dans une AUTRE session : « Human Space » → « Team Space » (page ID inchangé `39124f2fef0c811599edec1c18b44b20`) et option Executor « 🧑 Human » → « 🧑 Team Member ». Vérifié par `notion-fetch` (page = « 👩🏽‍💼 Team Space » ; ds Executor = « 🧑 Team Member »). Texte de la fiche aligné : anchor `[Team Space]` (URL inchangée, `replace_all`), « 🧑 Human » → « 🧑 Team Member » (`replace_all`), et dernière mention « human » de Common mistakes.
   c. Suppression de TOUS les tirets cadratins « — » (demande Karter) : remplacés par deux-points / point-virgule / parenthèses ; intro reformulée avec parenthèses.
   d. Section Related : ajout d'un lien cliquable direct vers l'index 📚 Knowledge (`39f24f2fef0c814eae67e7a829f3c8c2`). Les titres « Validate an agent's work » et « The Cockpit » restent en texte pour l'instant, à convertir en liens directs une fois ces fiches créées.
5. Mémoire projet : création `aftrsn-procedures-wiki.md` (conventions + avancement) + ligne d'index dans `MEMORY.md`.
6. Validation Karter : ✅ 17.07.2026.

## IDs clés
- Fiche créée : `3a024f2f-ef0c-81e6-8d24-db5a79a8630c`
- Wiki : page `39f24f2fef0c814eae67e7a829f3c8c2` / db `b118536be07242e1beee2373393fed6b` / ds `7f499c43-0dc2-4ad1-b23b-6412363ff4e2`
- Tasks ds `e50f931d-3f8d-483f-895a-c09408455128` ; template New Task `39e24f2fef0c802695c8ccb3accb22e0` ; Team Space page `39124f2fef0c811599edec1c18b44b20`

## Difficultés rencontrées
- Édition concurrente : Karter modifiait la même page au navigateur pendant la session, donc les search-and-replace « à l'aveugle » ont échoué (texte stocké différent de l'attendu).
- Un renommage structurel (Space + option Executor) est arrivé EN COURS de rédaction, rendant le vocabulaire de la fiche obsolète en direct.
- `update_content` exige un match exact de la chaîne cible.

## Solutions implémentées
- Re-fetch systématique de la page AVANT chaque correction, pour cibler la chaîne réelle.
- Vérification directe de la vérité terrain (fetch de la page Space et de la ds Tasks) avant d'aligner le vocabulaire, au lieu de se fier au seul message.
- Éditions par petits `update_content` ciblés (chirurgical) plutôt qu'un `replace_content` complet, pour ne pas écraser les éditions manuelles de Karter.

## Lessons learned
- Quand l'utilisateur édite en parallèle au navigateur : re-fetch juste avant d'écrire et préférer `update_content` à `replace_content`.
- Les IDs Notion sont stables au renommage : un lien par ID survit à un changement de titre ; seul le texte visible est à mettre à jour.
- Convention de navigation posée : le bloc Related doit devenir des titres-liens cliquables directs (à câbler dès que les cibles existent, dans les deux sens).
- Règle de style : zéro tiret cadratin « — » dans les fiches et les POS.
- Dépendance ouverte : le renommage 🧑 Human → 🧑 Team Member se propage depuis une autre session ; toute fiche doit employer le nouveau vocabulaire.
