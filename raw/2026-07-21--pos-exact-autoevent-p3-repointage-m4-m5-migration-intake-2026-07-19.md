---
type: raw
title: "POS-EXACT_AutoEvent-P3_repointage-M4-M5-migration-intake_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_repointage-m4-m5-migration-intake_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent P3 · Repointage M4/M5 sur les bases Content + migration & archivage Intake

**Date :** 19.07.2026 · **Marque :** AFTRSN · **Type :** POS-EXACT · **Statut :** ✅ FAIT, TESTÉ LIVE, VALIDÉ (Karter 19.07). Clôt la bascule de la chaîne sur « Contents & Metrics » (voir POS refonte M2 + Bible v3.11).

---

## 1. M4 `AFTRSN-WIX-EVENT-RUNNER` (`6XVotFtRAZ7BmLBk`, 20 nœuds) — repointé sur Website Content
- **`List copies`** : remplacé par HTTP POST `databases/1a0c078ddd454477b10c8de1d673c16b/query` (Website Content), body STATIQUE `{"filter":{"property":"Status","select":{"equals":"Approved"}},"page_size":100}`. **Nouveau comportement : gate par canal — M4 ne pousse vers Wix que du contenu `Approved`** (avant : toute copie avec du texte).
- **`Should build?`** : parse la réponse brute — titre contient ` · Event page`, `Short/Long description` (rich_text), `editionId` = `properties.Edition.relation[0].id` (fiable ici : ligne créée par NOTRE M2, cible vivante ; skip si vide = édition archivée, comportement voulu). Sort `{editionId, copyId, wixShort, wixLong}` → **`Prepare event` et tout le tail Wix inchangés**.
- **Preuve live** : exéc. `20767` (ligne To Validate → 0 item, gate bloque) · exéc. `20779` (ligne Approved → editionId + descriptions extraits, route `blocked` correcte [EP05 sans Event-type], **garde-fou de drainage** : pas de tâche « Wix event page » To Do → 0 écriture Wix/Notion). Chemin Wix réel = tail inchangé (prouvé 18-19.07) + re-prouvé au test final E2E.

## 2. M5 `AFTRSN-NEWSLETTER-TRACKER` (`HnX3llf1rW6qmlPb`, 6 nœuds) — repointé sur Newsletter Content
- **`List copies`** : URL → `databases/190dd20141e5468a81c7b0762e579832/query` (filtre `Approved` inchangé).
- **`Assemble brief`** : parse le titre canon `<edition> · <wave> · Newsletter` (wave = avant-dernier segment, edition = tout le début) + champs `Subject`/`Preview`/`Body`. Matching tâche↔copie par TITRE (édition+vague), inchangé côté tâche (`<wave> · Newsletter · <edition>`).
- **Preuve live (EP05)** : copie To Validate → tâche reste To Do ; copie **Approved** → tâche remplie (SUBJECT/PREVIEW/BODY + source) et passée To Validate. **État de test remis à zéro** ensuite (copie → To Validate, tâche → To Do + Notes vide) pour le replay du test final.

## 3. Migration & archivage Intake
- **Migré** : copie EP6 Announce → 3 lignes canon dans les bases Content (`EP06 - AFTRSN - HOME (Street Parade Edition) · Announce · Newsletter` / `· Announce · Instagram` / `· Event page`), **To Validate**, reliées à EP6. Non migrés : vieux draft Mini Café (contenu obsolète d'EP05, remplacé par les rows de test M2) + copies EP07-TEST (junk).
- **Archivé** : base ✍🏽 Intake (`709b816f…`) mise à la corbeille Notion (récupérable ~30 j) sur feu vert Karter. **Plus aucun runner ne la lit/écrit.**

## 4. État de la chaîne après bascule (tout ACTIF, scan 1 min)
M1 (checklist) → M2 (Iris → 3 bases Content, To Validate) → gate Karter PAR CANAL (Approved dans chaque base) → M4 (Website Content Approved → page Wix draft) · M5 (Newsletter Content Approved → tâche prête à coller) · M3 (visuels, inchangé). Metrics : bases prêtes, runners à construire (étape suivante).

## 5. Rappels
Matching par TITRE partout (relation lue en scan non fiable — sauf relation d'une ligne créée par notre propre runner sur cible vivante, avec skip-si-vide) · body JSON statique = OK inline ; body dynamique = nœud Code (`JSON.stringify`) · tester le gate sur ses DEUX faces + le garde-fou de drainage avant de valider.
