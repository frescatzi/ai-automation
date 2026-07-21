---
type: raw
title: "POS-LUMINA_Intendance-Drive-LUMINA-AI-DOCS_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_intendance-drive-lumina-ai-docs_2026-07-12.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-LUMINA — Intendance du Drive `LUMINA AI DOCS`
**Scope :** LUMINA OS · **Date :** 2026-07-12 · **Dossier :** `LUMINA AI DOCS` (`1rg1dzCpsI1LjOKmKEzaE0ZH83ij_i6Ot`)

## But
Maintenir la « boussole » documentaire fiable : une seule version à jour par doc (la Bible complète + les SPEC-BUILD/POS/PASSATIONS), zéro doublon de redépôt.

## Préconditions
1. Connecteur Google Drive autorisé sur **itiskarter@gmail.com** (create/read/search ; **pas de suppression**).
2. Claude in Chrome connecté au **même** compte (dossier sous `/u/3/`) pour tout dépôt lourd ou toute suppression.
3. Bible maîtresse locale `BIBLE_LUMINA-OS_360_2026-07-02.md` à jour des derniers addenda.

## Étapes (numérotées à partir de 1)
1. **Scanner** `LUMINA AI DOCS` par connecteur ; repérer homonymes et versions périmées (le redépôt connecteur crée des doublons).
2. **Identifier** chaque doublon par lecture + date + taille ; noter les IDs ; décider garder/supprimer (invariant : garder la version la plus récente et la plus complète).
3. **Fusionner** les addenda Bible en attente dans la maîtresse ; incrémenter l'en-tête de version (numérotation **à partir de 1**).
4. **Déposer** la Bible complète à jour : **upload navigateur** (fichier ~80 Ko, fidélité) → vrai `.md` (`text/markdown`).
5. **Supprimer** les doublons périmés par navigateur : ciblage **`data-id`**, vérif sélection par ID, **corbeille** (jamais vider).
6. **Vérifier** par connecteur : Bible unique en `text/markdown` ; doublons absents.
7. **Documenter** (POS-EXACT + GENERIQUE + POS) + **double-sauvegarde** (projet Claude + Drive) ; addendum Bible / tick Spec Build si pertinent.

## Vérification
- `BIBLE_LUMINA-OS_360_2026-07-02.md` : 1 exemplaire, `text/markdown`, taille attendue (82 780 o au 12-07, id `1OOFtKlEpQ1Nfrb0YNaN8G5bov1G0JOrG`).
- Un seul `SPEC-BUILD…GATEWAY-INTENT-ROUTER` (v0.3, `1jgJu6lzawHeZsmjCEl6TjK-MDPT6e4rH`).

## Rollback
- Corbeille Drive → Restaurer le fichier supprimé ; supprimer par ID un dépôt erroné.

## Difficultés rencontrées
- Connecteur Drive sans suppression ; navigateur initialement sur le mauvais compte ; input file absent du DOM (dialogue natif) ; fichiers homonymes ; décalage coordonnées clic/DOM.

## Solutions implémentées
- Dépôt + suppression par navigateur (compte propriétaire) ; connexion itiskarter (`/u/3/`) ; patch `click` + `file_upload` + `dispatch('change')` ; ciblage `data-id` + preuve de sélection ; conversion des coordonnées ; suppression en corbeille.

## Lessons learned (pièges figés)
- Connecteur Drive = **pas de delete** → navigateur (compte propriétaire) ou Karter.
- Gros `.md` fidèle = **upload navigateur** ; redépôt connecteur = **doublon** à nettoyer.
- **Cibler par `data-id`** et **prouver la sélection par ID avant toute suppression** ; suppression = **corbeille** réversible.
- Intendance recommandée **avant** toute reprise produit (M3), pour que la boussole reste fiable.

*POS-LUMINA — 2026-07-12. Double-sauvegardé : projet Claude « Claude Lessons » + Drive `LUMINA AI DOCS`.*
