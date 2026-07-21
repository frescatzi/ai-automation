---
type: raw
title: "POS-GENERIQUE_Intendance-dossier-Drive-nettoyage-doublons-et-redepot"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_intendance-dossier-drive-nettoyage-doublons-et-redepot.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GENERIQUE — Intendance d'un dossier Drive (nettoyer les doublons + redéposer un doc à jour)
**Scope :** réutilisable pour n'importe quel dossier Google Drive et n'importe quelle marque.

## But
Ramener un dossier Drive à « une seule version à jour par document » : fusionner/mettre à jour un document maître, le (re)déposer en vrai fichier (`.md` ou autre), et supprimer les versions périmées / doublons — sans jamais supprimer le mauvais fichier.

## Préconditions
- Connecteur Drive autorisé sur le compte **propriétaire** du dossier (lecture/écriture). ⚠️ La plupart des connecteurs Drive **ne suppriment pas**.
- Pour supprimer ou déposer un gros fichier : navigateur piloté connecté **au même compte propriétaire**.

## Étapes (numérotées à partir de 1)
1. **Scanner** le dossier (`search_files parentId='<folderId>'`) : lister les fichiers, repérer homonymes et versions périmées.
2. **Distinguer les doublons** en lisant le contenu / en comparant **date + taille** ; décider quelle version garder et lesquelles supprimer. Noter les **IDs** exacts.
3. **Mettre à jour le document maître** (fusion d'addenda, incrément de version dans l'en-tête) dans la source locale.
4. **Déposer le document à jour** :
   - Petit fichier → connecteur `create_file` avec `contentMimeType` explicite + `disableConversionToGoogleType=true` (garde le vrai type, évite la conversion en Google Doc). ⚠️ Le connecteur **crée un doublon** (ne remplace pas) → supprimer l'ancien ensuite.
   - Gros fichier / fidélité critique → **upload navigateur** : si l'`input[type=file]` n'existe pas (créé à la volée + dialogue natif), patcher `HTMLInputElement.prototype.click` pour capturer l'input et l'attacher au DOM, puis `file_upload` sur sa ref, puis `dispatch('change')`.
5. **Supprimer les versions périmées / doublons** (navigateur si le connecteur ne supprime pas) : cibler la ligne par **`data-id`**, **vérifier la sélection par ID** (`aria-selected`) avant d'agir, puis **corbeille** (réversible). Ne jamais vider la corbeille.
6. **Vérifier** via le connecteur : le doc à jour est unique + au bon type ; les doublons ont disparu du listing actif.
7. Produire la doc (POS-EXACT + GENERIQUE + POS) et double-sauvegarder ; addendum Bible / tick Spec Build si pertinent.

## Vérification
- Le doc maître apparaît **une seule fois**, au bon `mimeType`, taille attendue.
- Les IDs des doublons ne sont plus dans le dossier.

## Rollback
- Restaurer depuis la Corbeille (fenêtre de rétention Drive) ; supprimer un dépôt erroné par son ID.

## Difficultés rencontrées (typiques)
- Connecteur Drive **sans suppression**.
- Navigateur sur le **mauvais compte** Google (accès refusé).
- **Pas d'input file** dans le DOM (Drive le crée au clic + dialogue natif non pilotable).
- **Fichiers homonymes** (risque de mauvaise cible).
- **Décalage coordonnées** clic vs `getBoundingClientRect` (viewport CSS ≠ pixels du screenshot).

## Solutions implémentées (typiques)
- Suppression + gros dépôt via **navigateur** sous le compte propriétaire.
- Connexion du bon compte (le dossier se résout sous `/u/N/`).
- Patch `click` + capture de l'input + `file_upload` + `dispatch('change')`.
- Ciblage par **`data-id`** + preuve de sélection par ID avant action destructive.
- Conversion des coordonnées par le facteur d'échelle screenshot/viewport.

## Lessons learned (pièges figés)
- **Vérifier les capacités du connecteur avant de planifier** : create ≠ delete ≠ update.
- **Gros fichier fidèle = upload navigateur**, pas un round-trip par la conversation.
- **Toujours cibler par identifiant** (jamais par nom seul) et **prouver la sélection avant toute suppression**.
- **Suppression = corbeille** (réversible) ; ne jamais vider la corbeille.
- **Redépôt connecteur = doublon** → nettoyer l'ancien systématiquement.

*POS-GENERIQUE — procédure abstraite, réutilisable par dossier/marque.*
