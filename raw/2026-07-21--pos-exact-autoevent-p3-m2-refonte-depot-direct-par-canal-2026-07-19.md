---
type: raw
title: "POS-EXACT_AutoEvent-P3_M2-refonte-depot-direct-par-canal_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m2-refonte-depot-direct-par-canal_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent P3 · Refonte M2 — Iris dépose DIRECT dans les bases Content par canal

**Date :** 19.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Type :** POS-EXACT · **Statut :** ✅ CONSTRUIT, TESTÉ LIVE (2 vagues EP05), VALIDÉ (Karter, 19.07), ACTIF.

---

## 0. Contexte — la refonte « Contents & Metrics »
Karter a retravaillé l'architecture du contenu (validée en 2 itérations de maquette) : sous **📡 Contents & Metrics** (page, sous Operations), deux environnements avec la même séparation par plateforme :
- **📦 Content Pack** : `📤 Newsletter Content` · `📸 Instagram Content` · `🌐 Website Content` — le TEXTE (statut, lien publié). C'est ici qu'Iris dépose.
- **📊 Metrics** : `Newsletter Metrics` · `Instagram Metrics` · `Website Metrics` — les CHIFFRES (As-of), chaque ligne reliée à sa ligne Content + à l'édition. Wix = auto (à construire), Instagram = saisie manuelle à date figée.
L'ancienne base Content & Social (→ **✍🏽 Intake**, DB `709b816f…`, ids inchangés) sort de la structure ; **plus aucun runner d'éclatement nécessaire** : M2 écrit directement par canal. Intake sera archivée après repointage de M4/M5.

## 1. Ce qui a changé dans `AFTRSN-COPY-RUNNER` (`XQYvzTzFK2NLjDZz`, 18 nœuds)
**Tête inchangée** (scan 1 min → List tasks → Should draft? [« Copy & captions », To Do, 🤖 Agent] → Read the edition → Read line-up roster → Build the query → Draft the copy [Iris, entrée `query`] → Draft OK? → If draft OK). **Tail remplacé** :

~~`Write draft to Content & Social` → `Build body blocks` → `Append body blocks`~~ →
1. **`Write Newsletter Content`** (Notion create, DB `190dd201…`) : Title `{edition} · {wave} · Newsletter`, Edition (relation), Wave (text), **Status=To Validate**, Subject/Preview/Body, Campaign angle.
2. **`Write Instagram Content`** (create, DB `efb15b81…`) : Title `{edition} · {wave} · Instagram`, Caption, Hashtags, Campaign angle, Wave, Status=To Validate.
3. **`Build website query`** (Code) : construit `{filter:{property:'Title',title:{equals: edition+' · Event page'}},page_size:1}` en `JSON.stringify`.
4. **`Find Website row`** (HTTP POST `databases/1a0c078d…/query`, body `={{ $json.websiteQueryBody }}`, onError continue).
5. **`Website exists?`** (IF `($json.results||[]).length > 0`).
6. true → **`Update Website Content`** (update de la ligne unique : Short/Long description, Campaign angle, Wave, Status→To Validate) ; false → **`Create Website Content`** (create : Title `{edition} · Event page`, Edition, Wave, Status=To Validate, descriptions).
7. Les deux → **`Mark task To Validate`** (inchangé). Branche KO → `Mark task Blocked` (inchangée).

Toutes les valeurs viennent de `$('Draft OK?').first().json` (M2 = drainage par mutation-source, 1 tâche/tick ⇒ `.first()` OK). **Matching Website par TITRE** (jamais par relation — leçon M5).

## 2. ⚠️ Piège n8n CONFIRMÉ (et corrigé)
**Un objet JSON littéral dans une expression `={{ ... }}` échoue en « invalid syntax »** : les `}}` imbriqués de l'objet cassent le parseur d'expressions n8n. Symptôme vicieux : avec `onError: continueRegularOutput`, l'item d'erreur `{error:'invalid syntax'}` continue dans le flux → l'IF évalue 0 résultat → chemin `Create` à chaque vague → **doublons silencieux**. **Règle : tout body JSON se construit dans un nœud Code (`JSON.stringify`) et se référence par `{{ $json.xxx }}`** (pattern M3/M4, désormais canonique).

## 3. Preuve live (EP05 - AFTRSN - HOME, édition Done = gelée pour M1 ; tâches créées à la main)
- **Vague 1 « Announce »** (exéc. `20647`, 24 s) : 3 créations (Newsletter + Instagram + Event page, To Validate, reliées à EP05) + tâche → To Validate.
- **Vague 2 « Line-up reveal »** (exéc. `20668`, 23 s, après le fix §2) : 2 créations (Newsletter + Instagram vague 2) + **UPDATE de la MÊME page Event page** (id identique, descriptions vague 2, pas de doublon) → **upsert idempotent prouvé**. Line-up résolu en noms réels (Aïssa, Paqo, Bambona).
- Les lignes EP05 sont **conservées** : jeu de données du test final du moteur.

## 4. IDs
M2 `XQYvzTzFK2NLjDZz` (18 nœuds, actif, Error-WF Sentinel `xzaH0uWy0idKVphF`) · Newsletter Content DB `190dd20141e5468a81c7b0762e579832` · Instagram Content DB `efb15b81f4454142b32a2b6cd9a1ca91` · Website Content DB `1a0c078ddd454477b10c8de1d673c16b` · Intake DB `709b816fe2f84dbf90ee597803a11158` (encore lue par M4/M5) · Editions ds `61418ed4…` · EP05 `39e24f2f-ef0c-812a-b3bb-e8636418a482` · cred Notion `sCsj206WVkNxO0Jl`.

## 5. Reste (chantier Contents & Metrics)
Repointer **M4** (List copies : Intake → Website Content, gate Approved) → repointer **M5** (Intake → Newsletter Content) → runners **métriques Wix** → **archiver Intake**. Voir mémoire `aftrsn-contents-and-metrics`.
