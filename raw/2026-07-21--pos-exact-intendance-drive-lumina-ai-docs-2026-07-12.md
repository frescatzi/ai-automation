---
type: raw
title: "POS-EXACT_Intendance-Drive-LUMINA-AI-DOCS_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_intendance-drive-lumina-ai-docs_2026-07-12.md"
captured: 2026-07-21
vault: brands
brand: LUMINA
immutable: true
---

# POS-EXACT — Intendance du Drive `LUMINA AI DOCS` (nettoyage + Bible à jour)
**Date :** 2026-07-12 · **Scope :** LUMINA OS · **Exécutant :** Claude (Cowork) · **Compte Drive :** itiskarter@gmail.com

## But
Fiabiliser la « boussole » documentaire : une seule version à jour par doc dans `LUMINA AI DOCS` (`1rg1dzCpsI1LjOKmKEzaE0ZH83ij_i6Ot`). Concrètement (12-07) : fusionner les addenda Bible v2.1 + v2.2 dans la Bible maîtresse et la déposer en `.md`, puis supprimer le doublon `SPEC-BUILD…GATEWAY-INTENT-ROUTER` périmé.

## Préconditions
- Connecteur Google Drive autorisé en tant qu'**itiskarter@gmail.com** (lecture/écriture, mais **PAS de suppression** — voir Difficultés A).
- Navigateur Claude in Chrome connecté au **même compte** itiskarter@gmail.com (le dossier se résout sous `/u/3/`).
- Bible maîtresse locale `BIBLE_LUMINA-OS_360_2026-07-02.md` présente dans le projet Claude, à jour jusqu'à l'addendum v2.0.

## Étapes réalisées (numérotées à partir de 1)
1. **Scan** du dossier via connecteur (`search_files parentId='1rg1dzCpsI1LjOKmKEzaE0ZH83ij_i6Ot'`) → 5 sous-dossiers + ~32 `.md` racine listés.
2. **Doublon confirmé** : deux `SPEC-BUILD_LUMINA-TELEGRAM-GATEWAY-INTENT-ROUTER_2026-07-08.md` — lecture des deux :
   - `1x-n4Exp6GY_CkzQONTmi3bCehNYguh50` = **v0.2** (08-07, 18 322 o, « Command Bot »/`LuminaCCbot`, Bible v1.4) → **périmé**.
   - `1jgJu6lzawHeZsmjCEl6TjK-MDPT6e4rH` = **v0.3** (11-07, 21 454 o, « Assistant Bot »/`luminaLyraBot`, item business→Maestro, Bible v2.0) → **à conserver**.
3. **Fusion Bible** (local) : ajout des addenda **v2.1** (orchestration Maestro/latence — Haiku 4.5 + vérificateurs gpt-4o-mini) et **v2.2** (état de conversation `conversation_state`) en fin de `BIBLE_LUMINA-OS_360_2026-07-02.md` ; en-tête ligne 2 passé à **Version 2.2 · 2026-07-12**. Taille finale **82 780 o**.
4. **Dépôt Bible sur Drive** via **Claude in Chrome** (le connecteur exclu pour un fichier de 83 Ko — voir Difficultés) :
   - `Nouveau` → l'input `type=file` n'existe pas dans le DOM (créé à la volée + dialogue natif). Neutralisé par patch `HTMLInputElement.prototype.click` (capture l'input, l'attache au DOM, supprime l'ouverture native).
   - Clic « Importer un fichier » → input capturé (`window.__stashedFileInput`, `multiple`, connecté).
   - `file_upload` sur la ref de cet input → `BIBLE_LUMINA-OS_360_2026-07-02.md` (81 Ko). `dispatchEvent('change')` → toast « 1 importation terminée » ✓. Patch `click` restauré.
   - Résultat : id **`1OOFtKlEpQ1Nfrb0YNaN8G5bov1G0JOrG`**, `mimeType=text/markdown`, 82 780 o (vrai `.md`, non converti).
5. **Suppression du doublon v0.2** via Chrome : ligne identifiée par **`data-id`** en DOM (jamais au nom seul, homonymes), sélection vérifiée (`aria-selected` = uniquement `1x-n4Exp6GY…`, « 8 juil. | 18 Ko »), puis icône **corbeille** de la barre d'outils → déplacé vers la corbeille (réversible).
6. **Vérification** (connecteur) : Bible v2.2 unique et en `text/markdown` ; un seul `SPEC-BUILD…GATEWAY` restant (v0.3). Doublon absent du listing actif.

## Vérification
- `search_files` : `BIBLE_LUMINA-OS_360_2026-07-02.md` = 1 exemplaire, 82 780 o, `text/markdown`, id `1OOFtKlEpQ1Nfrb0YNaN8G5bov1G0JOrG`.
- `1x-n4Exp6GY_CkzQONTmi3bCehNYguh50` absent du dossier (corbeille) ; `1jgJu6lzawHeZsmjCEl6TjK-MDPT6e4rH` présent.

## Rollback
- Restaurer le doublon : Drive → Corbeille → `SPEC-BUILD…07-08.md` (8 juil.) → Restaurer.
- Retirer la Bible déposée : supprimer l'id `1OOFtKlEpQ1Nfrb0YNaN8G5bov1G0JOrG`.

## Difficultés rencontrées
- **A.** Le connecteur Drive **ne supprime pas** (create/read/search seulement) → impossible de retirer le doublon par le connecteur.
- **B.** Le navigateur était connecté à un **autre compte Google** (avatar « F ») sans accès → « Une autorisation est nécessaire ». Impossible de saisir un mot de passe (interdit).
- **C.** À l'upload, **aucun `input[type=file]` dans le DOM** (Drive le crée au clic puis ouvre un dialogue natif non pilotable).
- **D.** **Deux fichiers homonymes** (`…07-08.md`) → risque de supprimer le mauvais.
- **E.** **Décalage coordonnées** clic vs `getBoundingClientRect` : viewport CSS 1440×665 vs screenshot 1568×724 (scale ≈ 1,089 ; dpr 2) → les premiers clics visaient la ligne du dessus.

## Solutions implémentées
- **A/Bible** : dépôt du gros `.md` par **upload navigateur** (fidélité parfaite, zéro round-trip) ; suppression par navigateur sous le compte propriétaire.
- **B** : Karter a connecté itiskarter@gmail.com ; le dossier s'est résolu sous `/u/3/`.
- **C** : patch `HTMLInputElement.prototype.click` (capture + attache l'input, supprime le dialogue natif), `file_upload` sur la ref, puis `dispatch('change')`.
- **D** : ciblage par **`data-id`** ; vérification `aria-selected` par ID **avant** toute suppression.
- **E** : conversion des coords via le facteur d'échelle ; **preuve de sélection par ID** avant l'action destructive ; suppression = **corbeille** (réversible), jamais vider la corbeille.

## Lessons learned (pièges figés)
- **Le connecteur Drive ne supprime pas** : toute suppression passe par le navigateur (compte propriétaire) ou par Karter en manuel.
- **Gros `.md` fidèle = upload navigateur** sur un input capturé, jamais un round-trip du contenu par la conversation.
- **Homonymes Drive** : cibler exclusivement par `data-id` ; distinguer les versions par date + taille.
- **Clic navigateur = espace pixel du screenshot** : convertir depuis `getBoundingClientRect` (× screenshot/innerHeight). **Avant une suppression, prouver la sélection par ID.**
- **Suppression = corbeille** (réversible) ; ne jamais vider la corbeille.

*POS-EXACT — 2026-07-12. Double-sauvegardé : projet Claude « Claude Lessons » + Drive `LUMINA AI DOCS`.*
