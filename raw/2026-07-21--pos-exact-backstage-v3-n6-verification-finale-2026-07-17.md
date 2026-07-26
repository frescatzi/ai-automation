---
type: raw
title: "POS-EXACT_Backstage-v3_N6-verification-finale_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_n6-verification-finale_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · N6 — Vérification bout-en-bout + clôture du build

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Rôle : marche à suivre exacte du milestone N6 (dernier milestone du build v3, spec §4) — création de tâches sonde, vérification du pont humain/agent des deux côtés, vérification des 10 bases et rollups, nettoyage. **Validé par Karter le 17.07.2026 → build v3 TERMINÉ.**

---

## 0. Préalables

- Connecteur Notion vérifié en début de session (`notion-fetch` id `self` → workspace AFTRSN OK).
- Boussoles lues : Spec Notion (`SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md`, amendements 2 & 3 en tête) + mémoire projet `aftrsn-backstage-v2-notion.md`.
- Décision de cadrage (P2-0, validée avant N6) : la tâche test « agent » est **conservée** après N6 — elle servira de tâche pilote pour l'intégration n8n (Hermès, périmètre = cette seule tâche).

## 1. Re-fetch de l'état réel (faisabilité)

`notion-fetch` sur `collection://e50f931d-3f8d-483f-895a-c09408455128` (data source Tasks) :
- Nom « Tasks » ✓ · Executor select `🧑 Human`/`🤖 Agent` ✓ · Status 5 options (To Do / In Progress / To Validate / Blocked / Done) ✓ · Category 8 options ✓ · relations Event (→ Editions `61418ed4-fb84-48fc-b372-247fbabe0c33`) et Venue ✓ · rollups Budget / Budget Used / Event Category / Event Date / Event Status ✓ · template New Task par défaut ✓.

## 2. Création des 3 tâches sonde (API)

`notion-create-pages`, parent `data_source_id = e50f931d-3f8d-483f-895a-c09408455128` :

| Tâche | Propriétés | ID créé |
|---|---|---|
| TEST N6 - Event task (Human) | Executor=🧑 Human · Status=To Do · Category=Daytime · Event=Mini Café `39e24f2fef0c812ab3bbe8636418a482` | `3a024f2f-ef0c-81ac-81ce-ea3dad8de777` |
| TEST N6 - Office task (Human) | Executor=🧑 Human · Status=To Do · Category=Admin · sans Event | `3a024f2f-ef0c-8170-8a61-e213dcf7bd7a` |
| TEST N6 - Agent task (pilot) | Executor=🤖 Agent · Status=To Validate · Category=Brand & Community · Event=Mini Café | `3a024f2f-ef0c-8130-9816-fc227ef9c3c3` |

## 3. Vérification des vues (API, query mode view ×5)

`notion-query-database-view` sur `https://www.notion.so/cd210f519f9a4d67a50ef6f50e851dc5?v=<view_id>` :

| Vue (view_id) | Attendu | Résultat |
|---|---|---|
| Board Tasks `39e24f2f-ef0c-8106-9293-000ce37a6613` | les 3 tâches sonde présentes | ✅ |
| Office `39e24f2f-ef0c-81f0-ab12-000c8881b427` | tâche office présente ; event et agent absentes | ✅ |
| 📋 Activity (Agents Space) `39f24f2f-ef0c-81d4-86fe-000ce67edf14` | tâche agent SEULE | ✅ |
| ⏳ To validate (Agents Space) `39f24f2f-ef0c-81f2-8914-000c22ff0461` | tâche agent SEULE | ✅ |
| ✅ Awaiting validation (Human Space) `39f24f2f-ef0c-8143-aa54-000c238a5ada` | tâche agent présente (+ EXAMPLE humaine To Validate) | ✅ |

**Le pont est validé** : une tâche Executor=🤖 Agent en Status=To Validate apparaît côté agent ET côté humain — zéro copie, vues filtrées sur la base unique. NB : la file ✅ Awaiting validation liste TOUT ce qui est To Validate (humain inclus) — c'est voulu, c'est la file founder.

## 4. Vérification des 10 bases + rollups

- `notion-fetch` page Operations `39124f2fef0c81efa298e09b37626bb8` : les 10 bases présentes et bien nommées — Editions, Post-Event Débriefs, Sponsors & Partners, Venues, DJs/Artists, Cities/Expansion, Budget & Finance, Content & Social, Marketing & KPI, Media Assets. ✅
- `notion-fetch` édition Mini Café `39e24f2fef0c812ab3bbe8636418a482` : relation Tasks inclut les tâches sonde liées ✓, 8 lignes Budget & Finance ✓, formules Budget Display / Budget Used / Profit résolvent ✓, Débrief lié ✓.
- Contrôle visuel navigateur sur la tâche event : rollups résolus (Budget 950/1370, Budget Used 0.69, Event Category Daytime, Days Until Due 0). ✅

## 5. Nettoyage (navigateur)

Corbeille des 2 tâches sonde **Human** uniquement (••• en haut à droite de la page → « Move to Trash ») :
1. `https://www.notion.so/3a024f2fef0c81ac81ceea3dad8de777` (event) → corbeillée.
2. `https://www.notion.so/3a024f2fef0c81708a61e213dcf7bd7a` (office) → corbeillée.
3. **La tâche agent `3a024f2f-ef0c-8130-9816-fc227ef9c3c3` est CONSERVÉE** (pilote P2).

Contre-vérification par re-query du board Tasks : seules restent la tâche pilote agent + les 4 tâches EXAMPLE. ✅

## 6. Clôture

- Validation Karter 17.07.2026 → **N6 coché, build Backstage v3 TERMINÉ (N1→N6)**.
- Bible 360° : PAS de mise à jour (milestone 100 % Notion, n8n intouché).
- Spec cochée + mémoire projet à jour + ce POS et son générique en triple sauvegarde.

---

## Difficultés rencontrées

1. **Menu ••• de Notion = toggle** : un clic pendant le chargement de la page n'ouvre rien, et re-cliquer referme le menu à peine ouvert — deux allers-retours perdus avant de trouver le rythme (un seul clic, puis `find` du menu item par accessibilité).
2. **Connecteur n8n au périmètre limité** (découvert lors de la vérif de connecteurs en ouverture) : il ne voit que les 9 workflows agents (Maestro, sous-agents, Hermes-Agent) — ni la Gateway Lyra, ni le Router, ni les workflows LUMINA-*. Sans impact sur N6, mais structurant pour la phase 2.
3. La vue ✅ Awaiting validation contient aussi des tâches humaines To Validate — surprise possible si on s'attend à une vue agent-only.

## Solutions implémentées

1. Séquence navigateur fiabilisée : naviguer → attendre le titre de page correct → UN clic sur ••• → `find("Move to Trash")` → clic sur la référence. Contre-vérification systématique par requête API après chaque corbeille.
2. Périmètre n8n documenté dans la passation/spec de phase 2 : le build P2 passera par le navigateur (import JSON / store Pinia), méthode éprouvée de la Bible.
3. Comportement re-vérifié contre le concept : la file founder = TOUT To Validate — conforme, rien à corriger.

## Lessons learned

1. **Une tâche sonde par cas de partition** (event/office/agent) suffit à vérifier tout le pont : chaque vue a un attendu binaire (présente/absente), la vérification devient une matrice mécanique.
2. **Garder la sonde « agent » comme pilote de la phase suivante** économise un cycle de création et garantit que le test P2 partira d'un objet déjà validé des deux côtés du pont.
3. La vérification API (query mode view) et la vérification visuelle sont complémentaires : l'API prouve les filtres, l'écran prouve les rollups/l'UX. Les deux ensemble = validation rapide et complète.
