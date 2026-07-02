---
type: raw
title: "Spec Obsidian-Workflow n8n"
source_url: "drive:1XzT5Zx4szaD5-oSXwQ7hY5ccy6vQ4I5H"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# Spec Obsidian — Workflow n8n `LUMINA — Archive — Raw→Drive`
*Objectif : après compilation `raw → wiki`, archiver le brut original dans Google Drive (rangé par
catégorie) et vider `raw/`. Source de vérité = `wiki/` ; le brut est conservé dans Drive comme filet
de sécurité.*

---

## Principe
**L'IA étiquette, n8n exécute.** Claude Code (`/ingest`) sait dans quelle catégorie il a rangé chaque
fichier ; il l'écrit dans `_archive_queue.json`. Ce workflow lit ce manifeste et fait le déplacement
physique vers Drive, puis supprime de `raw/`.

## Pré-requis
1. **Dossier Drive d'archive**, par ex. `Archive-Raw/`, avec un sous-dossier par catégorie.
   ⚠️ **DOIT être hors du dossier "Lumina Inbox"** surveillé par l'intake → sinon boucle infinie
   (archive ré-aspirée → re-poussée dans raw → re-compilée → ré-archivée…).
2. Le manifeste `_archive_queue.json` à la racine du repo (produit par `/ingest`), format :
   ```json
   [
     { "path": "raw/note-x.md", "category": "ai-automation", "processed_at": "2026-06-25" }
   ]
   ```

## Déclencheur
GitHub push sur le repo, **filtré sur les changements de `_archive_queue.json`** (ne pas se déclencher
sur tous les pushs). Alternative simple si le filtre est pénible : Schedule Trigger (ex. toutes les
heures).

## Chaîne de nodes
1. **Trigger** (GitHub push filtré, ou Schedule).
2. **Get a file** (GitHub) → lire `_archive_queue.json` (repo `frescatzi/ai-automation`, branch main).
3. **Code / Parse** → `JSON.parse` du manifeste → un item par fichier à archiver.
4. **Filtre "encore présent"** → pour chaque entrée, vérifier que le fichier existe toujours dans
   `raw/` (Get a file ; si 404 → déjà archivé, on skip). Rend le workflow **idempotent**.
5. **Get a file** (GitHub) → récupérer le contenu du fichier `raw/` (As Binary selon le type).
6. **Ensure folder** (Google Drive) → trouver/créer le sous-dossier `Archive-Raw/<category>`
   (Search folder → si absent, Create folder).
7. **Upload** (Google Drive) → déposer le fichier dans `Archive-Raw/<category>`.
8. **IF upload confirmé** (id de fichier Drive non vide) :
   - **[true]** → **Delete a file** (GitHub) sur le chemin `raw/...` (= vider raw/).
   - **[false]** → ne rien supprimer (on réessaiera au prochain run). Garde-fou identique à l'intake.
9. *(option)* **Nettoyer le manifeste** : réécrire `_archive_queue.json` sans les entrées traitées.
   Pas obligatoire — les entrées périmées sont déjà ignorées au step 4 (fichier absent = skip).

## Garde-fous (repris de l'intake, validés 2026-06-25)
- **Supprimer de `raw/` SEULEMENT après confirmation de l'upload Drive** → zéro perte de données.
- **N'archiver QUE les fichiers listés dans le manifeste** → les fichiers `raw/` pas encore compilés
  (absents du manifeste) sont laissés intacts pour le prochain `/ingest`.
- **Archive ≠ Inbox** → pas de boucle.
- La suppression dans `raw/` ne doit PAS déclencher les workflows pgvector/Notion (ceux-ci filtrent
  sur `wiki/`). À vérifier au branchement.

## Nommage / rangement
- Workflow : `LUMINA — Archive — Raw→Drive`, tags 🟢 prod · 🪝 webhook · 🔌 drive.
- Dossier n8n : infra LUMINA (pas brand-specific).

## Reste à décider au build
- Filtre push sur `_archive_queue.json` vs Schedule Trigger (simplicité vs réactivité).
- Faut-il nettoyer le manifeste (step 9) ou se reposer sur l'idempotence du step 4.
- Mapping exact `category` → arborescence Drive (1 niveau ? sous-thèmes ?).
