---
type: raw
title: "SOP_LUMINA-Archive-Raw-vers-Drive_n8n"
source_url: "drive:1xxNo_QM-3fqB2FW6z0J1xUdXYIpuDp6L"
captured: 2026-06-29
vault: ai-automation
brand: null
immutable: true
---

# POS — `LUMINA — Archive — Raw→Drive` (n8n)

> **Procédure Opérationnelle Standard** du workflow d'archivage. Référence d'exploitation : à quoi il sert, comment il tourne, comment le vérifier, quoi faire en cas de souci.
> **Version** : v1 (à plat dans `Archive-Raw`). **Statut** : actif. **Dernière maj** : 2026-06-29.

---

## 1. Objet

Archiver dans Google Drive les fichiers `raw/` déjà compilés en `wiki/` (listés dans `_archive_queue.json`), puis les supprimer de `raw/` **après confirmation de l'upload**. `wiki/` = source de vérité ; `raw/` transitoire ; Drive = filet de sécurité.

## 2. Déclenchement

- **Automatique** : cron `0 15 * * * *` (chaque heure à la minute 15).
- **Manuel** (test) : bouton **Test workflow** dans n8n.

## 3. Pré-conditions

- `_archive_queue.json` présent et à jour sur `main` (produit par `/ingest` + push Obsidian Git).
- Dossier `Archive-Raw` existant (ID `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`), **hors** de `Lumina Inbox`.
- Credentials n8n valides : `GitHub account`, `Google Drive account - Live`.

## 4. Déroulé (résumé)

1. **Schedule Trigger** (`0 15 * * * *`).
2. **Get a file** → lit `_archive_queue.json` (`frescatzi/ai-automation`, `main`, As Binary OFF).
3. **Code** → décode base64 + parse → 1 item/fichier (`path`, `category`, `processed_at`).
4. **Get raw file** → récupère `{{ $json.path }}` en binaire ; On Error = error output. **Success = à archiver / Error 404 = skip** (idempotence).
5. **Google Drive : Upload** → `data` vers `Archive-Raw` ; nom = `{{ $binary.data.fileName }}` ; On Error = error output.
6. **GitHub : Delete a file** (sur **Success** de l'upload uniquement) → supprime `{{ $('Code (décoder + parser le manifeste)').item.json.path }}` sur `main`.

## 5. Garde-fous (ne pas désactiver)

- Suppression branchée **uniquement** sur la sortie **Success** de l'upload.
- Seuls les fichiers du manifeste sont traités (les autres, ex. `README`, fichiers `vault: personal`, restent intacts).
- `Archive-Raw` doit rester **hors** de `Lumina Inbox`.
- Vérifier que la suppression dans `raw/` ne déclenche pas les workflows pgvector/Notion (ils filtrent sur `wiki/`).

## 6. Vérification d'un run sain

- **Run nominal** (nouveaux fichiers) : Node 4 Success = N, Error = 0 ; Upload = N ; Delete = N commits sur `main` ; fichiers visibles dans `Archive-Raw` ; `raw/` vidé des N fichiers.
- **Run à vide** (déjà archivés) : Node 4 Success = 0, Error = N ; Upload = 0 ; Delete = 0. **Comportement attendu, pas une erreur.**

## 7. Incidents & réponses

| Symptôme | Cause probable | Action |
|---|---|---|
| Node 2 = 404 | `_archive_queue.json` absent de `main` | Lancer `/ingest` + push Obsidian Git. |
| Tout en Error au Node 4 | Fichiers `raw/` absents (déjà archivés, ou pas poussés) | Normal si déjà archivés. Sinon vérifier que `raw/` est sur `main`. |
| Upload « no binary 'data' » | Un node intercalé a cassé le binaire | Garder l'Upload directement après `Get raw file` ; nom via `$binary.data.fileName`. |
| Suppression sans upload | Garde-fou contourné | Vérifier que Delete part bien de la sortie **Success** de l'Upload. |
| Doublons dans Drive | Re-run alors que `raw/` non vidé | Normal côté Drive ; nettoyer manuellement. L'idempotence agit côté `raw/`. |

## 8. Récupération

Un fichier supprimé de `raw/` est **récupérable** : copie dans `Archive-Raw` (Drive) **et** historique Git (`git log`/restauration du commit).

## 9. Coordonnées techniques

- n8n `n8n.aftersunpeople.com` v2.10.4 · dossier infra LUMINA · projet Personal.
- Repo `frescatzi/ai-automation` · branche `main`.
- `Archive-Raw` = `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk` · Inbox interdite = `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`.
- Secrets : uniquement dans n8n (jamais collés en conversation).

## 10. Évolutions prévues

- **v2** : rangement par catégorie `Archive-Raw/<category>/`.
- Nettoyage éventuel du manifeste (entrées périmées re-vérifiées à chaque run).
