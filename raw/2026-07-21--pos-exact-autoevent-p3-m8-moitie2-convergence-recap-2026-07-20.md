---
type: raw
title: "POS-EXACT_AutoEvent-P3_M8-moitie2-convergence-recap_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m8-moitie2-convergence-recap_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · M8 moitié 2 · Récap de convergence (AFTRSN-CONVERGENCE-RECAP)

Date : 20.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Moteur : n8n LUMINA OS (self-hosted Hetzner/Coolify, https://n8n.aftersunpeople.com) × Notion « The Backstage »
Statut : ✅ CONSTRUIT, TESTÉ LIVE, VALIDÉ (Karter), ACTIF. Clôture de M8 (moitié 2).

## 1. But
Quand tous les canaux d'une vague sont approuvés, produire un récap consolidé « prêt à diffuser » déposé dans une tâche dédiée `<vague> · Go-live · <édition>`, en `To Validate`. ZÉRO envoi/publication automatique : l'humain diffuse derrière son gate.

## 2. Décisions de cadrage (tranchées avec Karter, 20.07)
- **Où atterrit le récap** : une tâche `<vague> · Go-live · <édition>` dans la checklist (patron M5/M6 : la tâche = unité d'action humaine, rails To Do → To Validate, à côté des tâches canaux).
- **Qui pose la tâche** : M1 (Edition Watcher) en amont, comme les autres canaux ; M8b la consomme.
- **Signal de convergence** : les lignes Content de la vague au statut `Approved` ou au-delà (source unique, même gate que les trackers M5/M6). Le runner ré-assemble le récap depuis les lignes (self-contained).
- **Périmètre convergence** : lignes Newsletter + Instagram **de la vague** (celles qui existent) `Approved+` **ET** la page événement `<édition> · Event page` (Website Content, 1/édition) `Approved+` (précondition globale : on ne diffuse pas vers un événement dont la page n'est pas en ligne).
- **URL de la page événement dans le récap** : lien vers la page **Notion de l'Édition** (le lien Wix y figure déjà via `Wix draft URL` / `Event Link`), pas d'appel Wix. Lu depuis la relation `Event` de la tâche Go-live (aucun nœud en plus).
- **Rétroactivité** : mécanisme permanent = seulement les nouvelles vagues (logique anti-doublon de M1). Backfill ponctuel des tâches manquantes fait une fois (voir §7).

## 3. Le patch M1 (git92S6gDU1hGZ2b)
Nœud `Plan production tasks`, liste `jobs` : ajout de `"Go-live"` en fin de liste.
`["Copy & captions", "Flyer/visual", "Wix event page", "Newsletter", "Instagram post + story", "Go-live"]`
Effet : chaque **nouvelle** vague pose désormais aussi `<vague> · Go-live · <édition>` (Status To Do, Executor 🤖 Agent, relation Event). Non rétroactif (dédup par nom de tâche). M1 reste actif, aucun autre changement.

## 4. M8b · AFTRSN-CONVERGENCE-RECAP = `EmKjdNkEczjDfM8R` (7 nœuds, scan 1 min, Sentinel `xzaH0uWy0idKVphF`, cred Notion `sCsj206WVkNxO0Jl`)
Chaîne linéaire :
1. `Every minute` (scheduleTrigger 1 min).
2. `List Go-live tasks` (HTTP POST Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5/query`, filtre `Status = To Do` AND `Task contains "Go-live"`).
3. `List NL copies` (HTTP Newsletter Content `190dd20141e5468a81c7b0762e579832`, filtre Status ∈ {Approved, Scheduled, Sent}).
4. `List IG copies` (HTTP Instagram Content `efb15b81f4454142b32a2b6cd9a1ca91`, filtre Status ∈ {Approved, Scheduled, Posted}).
5. `List Web copies` (HTTP Website Content `1a0c078ddd454477b10c8de1d673c16b`, filtre Status ∈ {Approved, Live}).
6. `Check convergence + assemble` (Code) : indexe NL et IG par `edition||wave` (titre `<edition> · <wave> · Newsletter|Instagram`), Website par `edition` (titre `<edition> · Event page`). Pour chaque tâche Go-live To Do/Agent (parse `<vague> · Go-live · <édition>`), converge SI NL(vague) ET IG(vague) ET Web(édition) présents dans les jeux Approved+. Assemble les Notes (SUBJECT/PREVIEW newsletter, teaser + hashtags + visuel IG, lien Notion Édition, ordre de diffusion suggéré). Lien Édition = `https://www.notion.so/<id relation Event sans tirets>`. Garde-fou longueur < 1900 car.
7. `Fill Go-live task + To Validate` (Notion update, `onError continueRegularOutput`) : écrit Notes + Status `To Validate`.

Le récap NE re-duplique PAS les corps complets (body newsletter, caption entière, story) : ils sont déjà dans les tâches canaux remplies par M5/M6. Le Go-live pointe dessus (« Full body in task … », « Full caption + story in task … ») + résumé + ordre de diffusion.

## 5. Invariants respectés
Écrit uniquement Notes + Status=To Validate (jamais Approved/Done/Target). Rien ne sort (aucun appel d'envoi Wix/IG). Gate systématique. Matching par TITRE (jamais la relation lue en scan pour le matching ; la relation Event n'est lue que pour construire le lien Notion, usage d'affichage). Multi-items (parcours indexé, pas de `.first()` sur une liste). Auto-drainage : source = la tâche filtrée To Do → une fois remplie (To Validate) elle sort du scan.

## 6. Preuves live (EP05 - AFTRSN - HOME, terrain de test)
- **Gate fermé** (exéc. 24919) : tâche Go-live présente, Event page `To Validate` → `Check convergence` 0 item, aucune écriture.
- **Remplissage** (exéc. 24935) : Event page passée `Approved` → tâche Go-live `To Validate` avec récap complet.
- **Enrichissement + multi-items** (exéc. 28982) : les 2 vagues EP05 HOME (Announce + Line-up reveal) convergent, ligne EVENT PAGE = `Wix link is on the Edition page: https://www.notion.so/39e24f2fef0c812ab3bbe8636418a482`, 2 tâches remplies.
- Terrain remis en fixture : Event page `To Validate`, 2 tâches Go-live `To Do` + Notes vides.

## 7. Backfill ponctuel (20.07)
Constat : toutes les éditions sont gelées (EP02/03/05/06 Done, EP04 Cancelled, EP07 Live). Hors EP05 HOME (terrain propre), les autres tâches existantes sont des restes de test ou au nommage incohérent (`EP07 - TEST AUTO`, `EP05 - AFTRSN LONDON` relié à EP06, `EP03 - AFRTSN -` tronqué, EP04 relié à une page fantôme) : y créer des Go-live produirait des tâches qui ne matcheraient jamais le contenu. Seul groupe (édition, vague) propre à compléter : **EP05 HOME · Line-up reveal** → tâche `Line-up reveal · Go-live · EP05 - AFTRSN - HOME` créée. EP05 HOME · Announce Go-live existait déjà (créée au test). Décision en attente de Karter : Go-live sur les éditions passées réelles (EP02/03/06/07) et/ou nettoyage des tâches de test.

## 8. IDs
- n8n : M8b `EmKjdNkEczjDfM8R` · M1 Edition Watcher `git92S6gDU1hGZ2b` · Sentinel `xzaH0uWy0idKVphF` · cred Notion `sCsj206WVkNxO0Jl`.
- Notion : Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` (ds `e50f931d-3f8d-483f-895a-c09408455128`) · Newsletter Content `190dd20141e5468a81c7b0762e579832` · Instagram Content `efb15b81f4454142b32a2b6cd9a1ca91` · Website Content `1a0c078ddd454477b10c8de1d673c16b` · Editions DB `dad7f826891f4c2980e4183c59bac8b0` (ds `61418ed4-fb84-48fc-b372-247fbabe0c33`) · EP05 HOME `39e24f2f-ef0c-812a-b3bb-e8636418a482`.
- Tâches de test EP05 : Announce Go-live `3a224f2f-ef0c-81fa-b008-e8344ec5ae52` · Line-up reveal Go-live `3a324f2f-ef0c-81cc-bbc5-eefecfa4d3be`.
