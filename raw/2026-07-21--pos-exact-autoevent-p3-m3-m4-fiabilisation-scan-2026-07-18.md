---
type: raw
title: "POS-EXACT_AutoEvent-P3_M3-M4-fiabilisation-scan_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m3-m4-fiabilisation-scan_2026-07-18.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · Fiabilisation M3 + M4 en scan planifié (comme M1/M2)

**Date :** 18.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Auteur :** session build 18.07 · **Type :** POS-EXACT (marche à suivre exacte, reproductible)

**Objet :** basculer `AFTRSN-VISUAL-TRACKER` (M3) et `AFTRSN-WIX-EVENT-RUNNER` (M4) des triggers Notion (Added/Updated) vers un **scan planifié 1 min + requête d'état**, pour éliminer la dépendance au trigger « Updated » (dé-dup par page → une reprise ne re-déclenche pas). Build 100 % par API n8n-mcp. Voir le POS-GÉNÉRIQUE associé pour le patron réutilisable (Difficultés · Solutions · Lessons).

---

## 0. Boussoles
`SPEC-BUILD_Automatisation-Evenementielle-P3` · LUMINA Bible 360° (addendum **v3.8**) · mémoire projet (`aftrsn-m3-visual-tracker`, `aftrsn-m4-wix-event`). Leçon fondatrice réutilisée : **trigger Notion « Updated » NON FIABLE ; utiliser scan planifié + requête d'état (patron M1/M2).**

## 1. Point de fond commun aux deux (à comprendre avant d'agir)
En scan planifié, le nœud « lister » (`getAll`) renvoie **toutes** les lignes à chaque tick, et le nœud Code « faut-il agir ? » boucle sur `$input.all()` et peut émettre **plusieurs** items éligibles dans le même tick. Or beaucoup de nœuds aval historiques lisaient le contexte via `$('NœudAmontMulti').first()` — ce qui, en mode trigger (1 item), était correct mais devient **faux en multi-items** : seul le 1ᵉʳ item est traité, les suivants sont perdus.

Deux régimes différents selon que le runner **mute ou non** l'objet qui l'éveille :
- **M2 (référence)** mute la tâche (`Status → To Validate`) → la liste filtrée `To Do` **rétrécit** → `.first()` traite 1 item/tick et se **draine** tout seul. C'est correct, on n'y touche pas.
- **M3** ne mute **jamais** la tâche (idempotence par requête d'existence) → la liste **ne rétrécit pas** → `.first()` **affame** toutes les vagues sauf la première. ⇒ M3 exige un vrai traitement **multi-items**.
- **M4** mute la tâche « Wix event page » (`→ To Validate`) → se draine comme M2, MAIS son scan est piloté par la **copie** (jamais mutée) → sans garde-fou il re-PATCHerait Wix chaque minute, et une copie drainée en tête masquerait une copie suivante. ⇒ M4 exige un **garde-fou de drainage** + multi-items sur les 2 nœuds de tête.

---

## 2. M3 — `AFTRSN-VISUAL-TRACKER` (`sJQ9Usa18kceEjlt`)

### 2.1 Bascule scan (opérations n8n `update_partial_workflow`)
1. **removeNode** `Notion Trigger - Task Added`
2. **removeNode** `Notion Trigger - Task Updated`
3. **removeNode** `Read the task`
4. **addNode** `Every minute` = `scheduleTrigger` (typeVersion 1.3), `rule.interval = [{field:minutes, minutesInterval:1}]`
5. **addNode** `List tasks` = node Notion `getAll` (`returnAll:true`) sur **Tasks DB** `cd210f519f9a4d67a50ef6f50e851dc5`, cred Notion `sCsj206WVkNxO0Jl`
6. **addConnection** `Every minute → List tasks`
7. **addConnection** `List tasks → Should track?`

Le code de `Should track?` est **inchangé** (il bouclait déjà sur `$input.all()` et sortait `{taskPageId, taskName, wave, editionId}`). Tout l'aval (Read the edition → Build asset + check → Check existing flyer → Create if new? → Create Media Assets row) reste connecté.

### 2.2 Durcissement multi-items (2 nœuds Code) — OBLIGATOIRE pour M3
Parce que M3 ne mute pas la tâche, on remplace `.first()` par un **parcours indexé** (l'ordre est préservé : `Read the edition[i]` ↔ `Should track?[i]`, etc.).

- **`Build asset + check`** : boucler sur `$input.all()` (les éditions), `ctx = $('Should track?').all()[i].json`, pousser 1 item par tâche avec `pairedItem:{item:i}`.
- **`Create if new?`** : boucler sur `$input.all()` (les réponses de `Check existing flyer`), `meta = $('Build asset + check').all()[i].json` ; `continue` si la requête d'existence renvoie ≥1 résultat (déjà présent), sinon pousser `meta`.

### 2.3 Preuve live (avant/après, EP07 a 2 tâches flyer To Do : Announce + Line-up reveal)
- **Avant** (exéc. 23:02, structure scan mais ancien `.first()`) : `Should track?` sort **2** → `Build asset + check` sort **1** (Announce affamé). ❌
- **Après** (exéc. 23:04, `16174`) : `Should track?` **2** → `Build asset + check` **2** (les 2 vagues, `pairedItem` 0 et 1) → `Create if new?` **0** (les 2 lignes existent déjà) → **0 doublon**. ✅
- Validation workflow n8n : 8 nœuds, 1 trigger, 0 erreur, graphe publié = version scan.

---

## 3. M4 — `AFTRSN-WIX-EVENT-RUNNER` (`6XVotFtRAZ7BmLBk`)

### 3.1 Bascule scan (opérations)
1. **removeNode** `Copy Trigger - Added`
2. **removeNode** `Copy Trigger - Updated`
3. **removeNode** `Read the copy`
4. **addNode** `Every minute` = `scheduleTrigger` 1 min
5. **addNode** `List copies` = Notion `getAll` (`returnAll:true`) sur **Content & Social DB** `709b816fe2f84dbf90ee597803a11158`, cred `sCsj206WVkNxO0Jl`
6. **addConnection** `Every minute → List copies`
7. **addConnection** `List copies → Should build?`

`Should build?` inchangé (bouclait déjà, sort `{editionId, copyId, wixShort, wixLong}`). Vérifié : le `getAll` renvoie bien les propriétés `property_wix_short_description` / `property_wix_long_description`.

### 3.2 Multi-items + GARDE-FOU DE DRAINAGE (2 nœuds Code de tête)
- **`Prepare event`** : boucler sur `$input.all()` (les éditions), `ctx = $('Should build?').all()[i].json`, calculer `route` (create/update/blocked) et `taskQueryBody` par item, pousser avec `pairedItem:{item:i}`.
- **`Attach task`** : boucler sur `$input.all()` (les réponses de `Find Wix task`), `prep = $('Prepare event').all()[i].json`, extraire `taskPageId` depuis `results[0].id` ; **`if(!taskPageId) continue;`** ⇒ si aucune tâche « Wix event page » en **To Do**, l'item est **écarté** (drainage). Sinon pousser.

`Find Wix task` interroge déjà la Tasks DB avec `Status = To Do` : c'est le signal de drainage. **Tout le tail (`Is update?` / `Build update` / `Update Wix event` / `Buildable?` / `Build event` / `Create Wix event` / `Parse create` / `Write Wix refs` / `Mark task To Validate(+upd)` / `Mark task Blocked`) reste STRICTEMENT INCHANGÉ** : il mute la tâche → il se draine une tâche/tick comme M2, et les `.first()` du tail restent valides (≤1 tâche actionnable réelle par tick).

### 3.3 Preuve live (EP07 : `Wix event ID = a3a02669-…`)
- **Croisière** (exéc. 23:13, `16210`) : 6 copies lues, 5 éligibles, `Find Wix task` × 5 = `results:[]` (aucune tâche To Do) → `Attach task` **0** → **aucun nœud d'écriture Wix exécuté** (pas de re-PATCH). ✅
- **Chemin actif** (test contrôlé : tâche « Line-up reveal · Wix event page » remise **To Do**) → exéc. 23:15 (`16222`) : `Attach task` laisse passer → `route=update` → `Update Wix event` **PATCH** OK, réponse **`status: DRAFT`** (rien de publié) → `Mark task To Validate (upd)` remet la tâche **To Validate**. ✅
- **Re-drainage** (exéc. 23:16, `16228`) : tâche de nouveau To Validate → `Attach task` **0** → **pas de re-PATCH**. ✅
- Validation workflow n8n : 18 nœuds, 1 trigger, 0 erreur, graphe publié = version scan.

---

## 4. État final
- M1, M2, **M3, M4** : tous en **scan planifié 1 min**. Zéro dépendance au trigger Notion « Updated » sur toute la chaîne événementielle.
- Invariants respectés : aucun automate n'écrit Done/Approved ; rien de publié (Wix reste `draft`) ; schémas inchangés ; secrets = Karter.
- EP07 laissé dans son état initial (tâche reveal revenue To Validate d'elle-même après le test).

## 5. Reste / à faire (hors périmètre de cette étape)
- Résidu Wix `shortDescription` (envoyée, persistance V3 à confirmer) ; chemin **Paid/EXTERNAL** non testé sur vraie édition payante.
- **Nettoyage EP07 - Event TEST-AUTO** (édition + tâches + copies + draft Wix `a3a02669-…`) en attente du feu vert de Karter.
- Suite moteur : **M5 Newsletter** (Wix natif, brouillon, gate envoi).
