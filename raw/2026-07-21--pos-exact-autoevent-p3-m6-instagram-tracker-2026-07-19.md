---
type: raw
title: "POS-EXACT_AutoEvent-P3_M6-instagram-tracker_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m6-instagram-tracker_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent P3 · M6 — Instagram Tracker (`AFTRSN-INSTAGRAM-TRACKER`)

**Date :** 19.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Type :** POS-EXACT (procédure exacte, reproductible) · **Statut :** ✅ CONSTRUIT, TESTÉ LIVE, VALIDÉ (Karter 19.07), ACTIF.

---

## 0. Résumé
`AFTRSN-INSTAGRAM-TRACKER` (`OjNFapHb4c0sNsCH`, 6 nœuds, scan 1 min) transforme la case de checklist « Instagram post + story » d'une vague en un **livrable prêt à poster** : feed (caption + hashtags approuvés) + story dérivée + visuel, assemblé dans le champ `Notes` de la tâche, puis passe la tâche `To Do → To Validate`. **Ne publie rien.** Publication manuelle par Karter derrière le gate (compte sensible, décision D2).

## 1. Pourquoi un tracker
Décision D2 : Instagram = programmation + post manuel (compte sensible ; pas d'API Meta de publication). Même patron que M3 (Canva) et M5 (Newsletter) : l'automate prépare et suit, l'humain poste. POS-GÉNÉRIQUE de référence = `runner-tracker-consommer-tache-checklist-remplir-livrable-pret-a-coller-gate-humain` (déjà écrite pour M5, validée ici sur un 2ᵉ canal).

## 2. Runner (état ACTIF)
- **ID** `OjNFapHb4c0sNsCH` · **nom** `AFTRSN-INSTAGRAM-TRACKER` · **6 nœuds** · Error-WF Sentinel `xzaH0uWy0idKVphF` · scan planifié 1 min.
- **Chaîne :** `Every minute` → `List IG copies` (HTTP) → `List tasks` (HTTP) → `Should post?` (Code) → `Assemble brief` (Code) → `Fill task + To Validate` (Notion update).

### 2.1 List IG copies (HTTP POST)
`https://api.notion.com/v1/databases/efb15b81f4454142b32a2b6cd9a1ca91/query` (Instagram Content) · cred `notionApi` `sCsj206WVkNxO0Jl` · header `Notion-Version: 2022-06-28` · body :
```json
{"filter":{"or":[{"property":"Status","select":{"equals":"Approved"}},{"property":"Status","select":{"equals":"Scheduled"}},{"property":"Status","select":{"equals":"Posted"}}]},"page_size":100}
```
→ gate copie = classes post-approbation. (Statuts IG Content : Draft / To Validate / Approved / Scheduled / Posted.)

### 2.2 List tasks (HTTP POST)
`https://api.notion.com/v1/databases/cd210f519f9a4d67a50ef6f50e851dc5/query` (Tasks) · même cred/header · body :
```json
{"filter":{"and":[{"property":"Status","select":{"equals":"To Do"}},{"property":"Task","title":{"contains":"Instagram post + story"}}]},"page_size":100}
```

### 2.3 Should post? (Code) — parse sans relation
Pour chaque tâche : `title` = `Task.title[].plain_text`, `status`, `executor`. `parts = title.split(' · ')`. Éligible si `parts.length>=3 && parts[1]==='Instagram post + story' && status==='To Do' && executor.indexOf('Agent')!==-1`. Émet `{taskPageId, taskName, wave=parts[0], editionName=parts.slice(2).join(' · ').trim()}`.
Titre tâche canon = `<wave> · Instagram post + story · <edition>` (posé par M1).

### 2.4 Assemble brief (Code) — matching par TITRE
Indexe les copies IG par clé `editionName||wave` extraits du titre `<edition> · <wave> · Instagram` (`last='Instagram'` ; `wave` = avant-dernier segment ; `edition` = reste). Exige `Caption` non vide. Pour chaque tâche, matche par `editionName||wave`. Sans copie ⇒ **skip** (tâche reste To Do). Construit `Notes` :
- **FEED** = `Caption` + `Hashtags` tels quels.
- **STORY** (dérivée) = teaser (1ʳᵉ ligne de la `Caption`, sinon `Campaign angle`) + `→ Link in bio` + rappel « texte minimal, même visuel ».
- **VISUAL** = `Visual link` (url de la ligne) si présent, sinon rappel « Attach the approved visual for this wave (Media Assets). »

### 2.5 Fill task + To Validate (Notion update, `onError: continueRegularOutput`)
`operation=update`, `pageId` url `=https://www.notion.so/{{ $json.taskPageId }}`, `Notes|rich_text` = notesText, `Status|select` = `To Validate`.

## 3. Décisions Karter (19.07)
- Feed servi tel quel + **story dérivée auto** (teaser = 1ʳᵉ ligne de la caption). Pas de champ Story dédié, pas de changement de schéma, Iris non touché.
- Visuel lu **directement dans `Visual link`** de la ligne IG Content (mapping Media Assets → Notion en amont) → **aucune requête n8n supplémentaire** ; si vide → ligne de rappel.
- Gate = copie post-approbation (Approved/Scheduled/Posted), aligné M5.

## 4. Drainage & idempotence
Source mutée = la **tâche** (To Do→To Validate) ; `List tasks` filtre `To Do` ⇒ pas de reprise (auto-drainage). **Multi-items** (toutes les tâches IG éligibles/tick). Copie absente ou non approuvée ⇒ tâche **reste To Do**, réessayée.

## 5. Preuve live (19.07)
- **Gate fermé** (exéc. `21701`) : 3 tâches IG To Do détectées (EP05 + 2× EP07), copie EP05 en To Validate + EP07 sans copie → `Assemble brief` sort **0** → rien rempli. ✓
- **Gate ouvert** (exéc. `21707`) : copie EP05 Announce IG → `Approved` → tâche EP05 remplie (FEED+STORY+VISUAL) → `To Validate` ; EP07 restent To Do. ✓
- Terrain remis à zéro : M6 réactivé (validé Karter), copie EP05 Announce IG → To Validate, tâche de test EP05 → To Do + Notes vidées (tag `demo`, replay E2E).

## 6. IDs
Instagram Content DB `efb15b81f4454142b32a2b6cd9a1ca91` · Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` · Media Assets DB `648cdeb7-bd6e-4113-9a26-dcf66a3271f6` (source des visuels, mappés dans `Visual link` côté Notion) · cred Notion `sCsj206WVkNxO0Jl` · Sentinel `xzaH0uWy0idKVphF` · workflow `OjNFapHb4c0sNsCH`.

## 7. Suite
M8 diffusion + KPI (Marketing & KPI ds `f42933a8-…`). M9 clôture + test final E2E sur EP05. Instagram Metrics = manuel.
