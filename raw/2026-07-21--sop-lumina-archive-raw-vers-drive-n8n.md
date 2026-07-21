---
type: raw
title: "SOP_LUMINA-Archive-Raw-vers-Drive_n8n"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/sop_lumina-archive-raw-vers-drive_n8n.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — `LUMINA — Archive — Raw→Drive` (n8n)

> **Procédure Opérationnelle Standard** du workflow d'archivage. Référence d'exploitation : à quoi il sert, comment il tourne, comment le vérifier, quoi faire en cas de souci.
> **Version** : v2 (rangement par catégorie, dossiers figés + repli racine). **Statut** : actif/publié (`LUMINA-INTAKE-ARCHIVE/RAW→DRIVE`, dossier `LUMINA-01-INTAKE`). **Dernière maj** : 2026-06-29.

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
3. **Code** → décode base64 + parse → 1 item/fichier (`path`, `category`, `processed_at`) **+ `folderId`** via `FOLDER_MAP` (catégorie → dossier ; repli racine `Archive-Raw`).
4. **Get raw file** → récupère `{{ $json.path }}` en binaire ; On Error = error output. **Success = à archiver / Error 404 = skip** (idempotence).
5. **Google Drive : Upload** → `data` vers **le dossier de catégorie** `{{ $('Code (décoder + parser le manifeste)').item.json.folderId }}` ; nom = `{{ $binary.data.fileName }}` ; On Error = error output.
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
| Purge `/sync` qui se trompe / `raw/` local incohérent avec Drive | Clone local **en retard sur `main`** (n8n écrit sur `main` sans toucher le local) | **`git pull` d'abord** ; le garde-fou de sync de la purge refuse d'agir si local ≠ `origin/main`. Voir §11. |

## 8. Récupération

Un fichier supprimé de `raw/` est **récupérable** : copie dans `Archive-Raw` (Drive) **et** historique Git (`git log`/restauration du commit).

## 9. Coordonnées techniques

- n8n `n8n.aftersunpeople.com` v2.10.4 · dossier infra LUMINA · projet Personal.
- Repo `frescatzi/ai-automation` · branche `main`.
- `Archive-Raw` = `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk` · Inbox interdite = `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`.
- Secrets : uniquement dans n8n (jamais collés en conversation).

## 10. Dossiers de catégorie (v2 — vocabulaire contrôlé)

`/ingest` classe chaque fichier dans **une seule** des **7 catégories canoniques** (liste fermée imposée dans `ingest.md` step 8, référencée par `sync.md`). Sous `Archive-Raw` (`16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`), table figée `catégorie → ID` dans le node Code :

| Catégorie | ID |
|---|---|
| `IA & LLM` | `1RZ0WmENcUPmXF4eekWASxX5AXyWUSGKU` |
| `Agents & MCP` | `1l8cyjWuW-HDv3DSMuvXkn9D7dwqQwoRB` |
| `Automatisation (n8n)` | `1KfqnFO18q67UcKEClvQihwoQlU_qG4To` |
| `Mémoire & Connaissance` | `1oc2Zd-W2HxNgqDrDwxMjSucXmdMexk63` |
| `Infrastructure` | `1r3kLYktqT5Vgi79syy2irslQKhCFmIEq` |
| `Architecture & Stratégie` | `1NtEvNjbhO6kjjytIPkNnJbzZLKAfxnmY` |
| `Méthodes & SOP` | `1KRoiAb8j6bcfEpRWYlzQ2mWj8OIqMGxY` |

**Catégorie hors-liste** → repli sur la racine `Archive-Raw` (fichier jamais perdu). **Ajouter une catégorie** : créer le dossier + ajouter sa ligne dans `FOLDER_MAP` (node Code) **et** dans la liste fermée de `ingest.md`. *(Une « Sphère B » agence/business est conçue mais non activée — à déployer à la création de l'agence.)*

## 11. Hygiène du manifeste & synchronisation (FAIT 2026-06-29)

- **Purge auto du manifeste** par `/sync` (étape PURGE de `ingest.md`) : retire les entrées dont le fichier n'est plus dans `raw/` (archivé), **conserve** les échecs (fichier encore présent), écrit `[]` si vide. **Durcie** par un garde-fou : ne purge que si le clone local est aligné sur `origin/main`.
- **Règle d'exploitation** : n8n écrit **directement sur `main`** (intake crée, archivage supprime) → le clone local se périme silencieusement → **toujours `git pull` avant tout travail local** (`/sync` le fait). `main` = vérité.
- **Checkout** : un seul clone confirmé (`~/Documents/Lumina AI/GitHub/ai-automation`) — pas de double checkout.

## 12. Évolutions futures

- Option : création des dossiers de catégorie à la volée (zéro maintenance) si la taxonomie devient mouvante.
