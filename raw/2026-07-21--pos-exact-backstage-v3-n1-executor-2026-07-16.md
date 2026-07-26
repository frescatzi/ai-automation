---
type: raw
title: "POS-EXACT_Backstage-v3_N1-executor_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_n1-executor_2026-07-16.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · N1 « Executor » (le contrat de données)

Date : 16.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **N1 FAIT & VALIDÉ par Karter le 16.07.2026** (captures à l'appui : pastilles sur le board + tâche test née en Human)
Référence : `SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md` (§4 N1) · `CONCEPT_Backstage-v3_Deux-Mondes_2026-07-16.md` (§3)

## Objectif

Créer la propriété select **Executor** sur la base Tasks — la propriété qui porte TOUTE la séparation Human/Agent de la v3 : `🧑 Human` (défaut via template) · `🤖 Agent`. Zéro copie de tâches : Agent Space (N3) ne sera qu'une vue filtrée `Executor = Agent` sur cette même base.

## Marche à suivre exacte

1. **Vérifier le schéma avant de toucher** : `notion-fetch` sur `collection://e50f931d-3f8d-483f-895a-c09408455128` (data source Tasks). Confirmé : pas d'Executor existant + ancestor-path = The Backstage `39124f2fef0c8172a961c8a57721ff0f` (le bon Backstage, AFTRSN HQ — pas le LUMINA).

2. **Créer le select** : `notion-update-data-source` sur `e50f931d-3f8d-483f-895a-c09408455128` avec le statement :
   ```sql
   ADD COLUMN "Executor" SELECT('🧑 Human':blue, '🤖 Agent':purple)
   ```
   Nouvelle colonne → PAS besoin de relister les options existantes (ce piège ne concerne que `ALTER COLUMN … SET SELECT` sur un select déjà en place). Résultat vérifié dans le schéma retourné : options 🧑 Human (blue) / 🤖 Agent (purple).

3. **Identifier les 4 tâches EXAMPLE** : `notion-query-data-sources` en SQL :
   ```sql
   SELECT url, "Task", "Status", "Category", "Executor" FROM "collection://e50f931d-3f8d-483f-895a-c09408455128"
   ```
   → 4 pages : Approve aftermovie final cut `39e24f2fef0c8171ab51d607b6af539c` · Renew website subscription `39e24f2fef0c8173b5e8d981a31341f5` · Pay artist fees `39e24f2fef0c818a9016c8e2091b15b7` · Follow up rooftop venue lead `39e24f2fef0c81b9a267cf144a08f8f1`.

4. **Taguer les 4 tâches** : 4× `notion-update-page` `update_properties` avec `{"Executor": "🧑 Human"}`. Re-query de contrôle → les 4 en 🧑 Human.

5. **Afficher Executor dans les vues clés** : `notion-fetch` sur la database `cd210f519f9a4d67a50ef6f50e851dc5` pour lister les vues, puis `notion-update-view` avec `SHOW` en relistant TOUTES les displayProperties (l'ordre passé = l'ordre affiché) :
   - Board **Tasks** `39e24f2f-ef0c-8106-9293-000ce37a6613` : `SHOW "Task", "Executor", "Due Date", "Venue", "Cost", "Event", "Owner"` → pastille en 2ᵉ position sur les cartes.
   - **Full Details** `39e24f2f-ef0c-8151-9638-000ceb263860` : idem, Executor inséré après Task.
   - **All** `1286496f-d056-408b-91b8-89b95c4e6f01` : rien à faire — cette vue affiche automatiquement toute nouvelle propriété.

6. **Défaut Human dans le template New Task** : `notion-update-page` `update_properties` `{"Executor": "🧑 Human"}` directement sur la page template `39e24f2fef0c802695c8ccb3accb22e0`. ✅ L'API a suffi (la spec prévoyait le navigateur) : un template Notion est une simple page de la collection. Vérifié par `notion-fetch` du template : `"Executor":"🧑 Human"`.

7. **Démo navigateur (Claude in Chrome)** : ouverture du board `notion.so/cd210f519f9a4d67a50ef6f50e851dc5?v=39e24f2fef0c81069293000ce37a6613` → pastilles 🧑 Human visibles sur les 4 cartes ; clic « New » → la tâche naît avec Executor = 🧑 Human pré-rempli ; tâche test mise à la corbeille (récupérable).

8. **Validation Karter** (16.07) → coche N1 dans la spec (3 dépôts) + ce POS + maj mémoire.

## Difficultés rencontrées

- La spec supposait le template New Task **non pilotable par API** (« probablement via navigateur »), héritage du piège connu « pas de DB templates via API ».
- Ordre d'affichage : `SHOW` remplace la liste complète des propriétés affichées — risque d'en perdre si on ne reliste pas tout.

## Solutions implémentées

- Test direct de `notion-update-page update_properties` sur la page template → fonctionne parfaitement ; vérifié par re-fetch. Zéro manipulation navigateur pour N1.
- À chaque `SHOW`, relisting complet des displayProperties existantes + Executor inséré en 2ᵉ position (juste après le titre).

## Lessons learned

- **Un template Notion est une page ordinaire de la collection** : on peut définir ses valeurs de propriétés par API (`update_properties`). Le piège « templates non pilotables par API » ne vaut que pour CRÉER/réordonner des templates, pas pour éditer leurs propriétés.
- **La vue « All » affiche automatiquement les nouvelles propriétés** — inutile de la toucher.
- **`ADD COLUMN` d'un select neuf est sans risque** (pas de relisting) ; le relisting complet n'est obligatoire que pour `ALTER COLUMN … SET SELECT` sur un select existant.
- **`SHOW` = liste exhaustive ordonnée** : l'ordre des propriétés dans le statement détermine l'ordre d'affichage (pratique pour placer la pastille Executor juste sous le titre des cartes).
- Le choix des couleurs (Human bleu / Agent violet) rend l'intervention agent repérable d'un coup d'œil dans les vues humaines — c'est le mécanisme « l'humain voit les agents sans quitter son monde » du concept v3.
