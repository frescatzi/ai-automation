---
type: raw
title: "POS-EXACT_AutoEvent-P3_M4-fix-paid-external_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m4-fix-paid-external_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent-P3 · M4 — Fiabilisation du chemin Paid / EXTERNAL (Wix Events V3)

**Date :** 19.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Périmètre :** `AFTRSN-WIX-EVENT-RUNNER` (`6XVotFtRAZ7BmLBk`) · **Type :** correctif + clôture de deux résidus M4.

---

## 1. Objet
Clore les deux résidus M4 laissés ouverts par la passation « moteur M1→M4 fiabilisé en scan » :
1. **Résidu `shortDescription`** (soupçonné « non persisté » côté Wix).
2. **Chemin Paid / EXTERNAL** codé mais jamais éprouvé live.

Résultat : le résidu `shortDescription` est un **artefact de vérification** (pas un bug) ; le chemin Paid était **réellement cassé** et est désormais **réparé et prouvé live**.

---

## 2. Résidu shortDescription — diagnostic & verdict
- **Symptôme historique :** après écriture, un GET renvoyait `shortDescription` vide.
- **Cause racine :** le GET de contrôle n'envoyait que `fields=TEXTS`. Or, dans Wix **Events V3**, le paramètre `fields` est un **tableau** (max 20) de projections, et :
  - `TEXTS` → renvoie `detailedDescription` (la description longue) ;
  - `DETAILS` → renvoie `shortDescription`, `mainImage`, `calendarUrls`.
  → `shortDescription` n'est **jamais** renvoyé sans `DETAILS`. On cumule : `?fields=TEXTS&fields=DETAILS`.
- **Preuve live (draft EP07 `a3a02669…`) :** avec `DETAILS`, `shortDescription` = « EP07, Zürich, daytime. SAM B and Mila lead the afternoon. Good people, great vibes. », `detailedDescription` présent, `status: DRAFT`.
- **Verdict :** **rien à corriger côté écriture.** La description longue vit sous la clé **`detailedDescription`** (pas `description`). Sur toute lecture de contrôle : demander `DETAILS`.

---

## 3. Chemin Paid / EXTERNAL — diagnostic
- **Test contrôlé :** édition EP07 basculée en `Event-type=Paid` + `Ticket URL`, `Wix event ID` vidé → route `create`. Exéc. runner en **erreur 400** au POST Wix :
  `event.registration.initialType value is required` (REQUIRED_FIELD) — alors que le body envoyait `initialType:"EXTERNAL"`.
- **Cause racine :** à la **création** V3, `registration.initialType` n'accepte **que** `RSVP` ou `TICKETING`. `EXTERNAL` y est **invalide** → traité comme absent → « required ». L'hypothèse antérieure « Paid = `initialType:EXTERNAL` » était fausse.
- **Le bon mécanisme (confirmé par la doc + prouvé sur un draft jetable) :** EXTERNAL s'obtient en **deux temps** :
  1. `POST` create `registration.initialType:'RSVP'`, `draft:true` ;
  2. `PATCH` `/events/v3/events/{id}` avec `{event:{id,registration:{type:'EXTERNAL',external:{url:ticketUrl}}}}`.
  Sur l'**update**, le champ mutable est `registration.type` (enum complet RSVP/TICKETING/EXTERNAL/NONE), pas `initialType` (qui « ne change jamais » et reste RSVP). `TICKETING` = billetterie **native Wix** (≠ le besoin AFTRSN, qui vend via un lien externe).

---

## 4. Correctif appliqué au runner (build par API MCP)
Workflow `6XVotFtRAZ7BmLBk`, désormais **20 nœuds** (version publiée `f87df6bf…`, validation 0 erreur).

**Nœuds modifiés (2 Code) :**
- **`Build event`** : crée **toujours** en RSVP draft (Free & Paid). Transporte `eventType`, `ticketUrl`, `needsExternal` (= Paid & ticketUrl non vide).
- **`Parse create`** : transporte `needsExternal`/`ticketUrl` et prépare `externalPatchBody` = `{"event":{"id":…,"registration":{"type":"EXTERNAL","external":{"url":…}}}}`.

**Nœuds ajoutés (2), branchés APRÈS la chaîne vérifiée `Mark task To Validate` (chaîne existante intacte) :**
- **`Needs external?`** (IF booléen sur `$('Parse create').first().json.needsExternal`).
- **`Set EXTERNAL registration`** (HTTP PATCH, cred `WIX AFTRSN-LUMINA` `2SjAjkAPXNNjUQYL`) sur la branche **true**.

**Propriété d'idempotence :** les refs Wix sont écrites (`Write Wix refs to edition`) **avant** le PATCH EXTERNAL → aucun doublon de création même si le PATCH échouait. Un échec du PATCH = visible via Sentinel + revue humaine (l'event reste un draft RSVP à valider). Free = **inchangé**.

---

## 5. Preuve live (re-test EP07)
Exéc. `16528` (scan planifié) — enchaînement complet, une seule création (drainage 4→1) :
`Attach task` (route=create, Paid) → `Build event` (RSVP, `needsExternal:true`) → `Create Wix event` → draft **`363b0a95-7355-4180-b8ff-3c45aea6e7a1`**, `registration.type: RSVP`, `status: DRAFT` → `Parse create` → `Needs external?` = **true** → `Set EXTERNAL registration` → **`registration.type: EXTERNAL`**, `external.url: …/tickets/ep07-test-auto`, **toujours `DRAFT`**, `shortDescription`/`detailedDescription` intacts.

---

## 6. Difficultés · Solutions · Lessons
- **Difficulté :** un GET « réussi » qui renvoie un champ vide fait croire à une non-persistance. **Solution :** vérifier la **projection** (`fields`) avant de conclure ; l'API peut masquer un champ pourtant écrit. **Lesson :** un succès HTTP ne prouve pas la lecture — vérifier avec la projection complète.
- **Difficulté :** une valeur d'enum invalide (`EXTERNAL` en `initialType`) est rapportée comme « champ requis manquant », pas « valeur invalide ». **Solution :** lire l'enum exact de l'API (create vs update peuvent différer). **Lesson :** create et update n'exposent pas forcément le même champ ni le même enum (`initialType` figé vs `type` mutable).
- **Difficulté :** en scan, forcer un `create` sur une édition ayant déjà **2 copies** éligibles produit **2 créations** (le `taskQueryBody` est identique par copie). **Solution :** neutraliser les copies surnuméraires pour un test create représentatif. **Lesson :** inoffensif en opération réelle (create = vague 1, 1 seule copie), mais à connaître.
- **Difficulté :** l'outil MCP Notion ne sait **pas** archiver/corbeiller (flags `archived`/`in_trash` ignorés). **Solution :** workflow n8n jetable → PATCH `archived:true` par API Notion, puis suppression du helper. **Lesson :** pour supprimer des pages Notion en masse, passer par l'API (pas le connecteur).

---

## 7. Nettoyage EP07 (fait)
Édition **EP07 - Event TEST-AUTO** purgée : **2 drafts Wix supprimés** (`a3a02669…` ancien RSVP orphelin + `363b0a95…` EXTERNAL du re-test) et **15 pages Notion archivées** (édition + 10 tâches + 2 copies Content & Social + 2 Media Assets) via helper n8n jetable. Réversible côté Notion (corbeille).

---

## 8. Invariants respectés
Wix reste **draft** (jamais publié). Aucun « Done/Approved » écrit par l'automate. Secrets = Karter uniquement. On appelle le runner LUMINA ; la modification a été explicitement validée. On a fixé la racine (pas de POST manuel de contournement). Build par API → montré → validé → documenté → on coche.
