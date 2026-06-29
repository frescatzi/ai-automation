---
type: raw
title: "HOWTO_archive-raw-vers-drive_workflow-n8n-LUMINA"
source_url: "drive:16dRNduOXL3tpma-RS9nGpacYG8JN3LwV"
captured: 2026-06-29
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre — Build du workflow n8n `LUMINA — Archive — Raw→Drive`

> **But du document** : avoir une clarté exacte et durable du build. Qui fait quoi, quand, où, comment et pourquoi — node par node, avec les challenges rencontrés, les solutions, et les leçons apprises.
> **Statut** : v1 construite, testée de bout en bout et idempotente (2026-06-29).
> **Environnement** : n8n self-hosté `n8n.aftersunpeople.com` (Hetzner/Coolify), v2.10.4. Dossier n8n : infra LUMINA. Projet : Personal.

---

## 1. Ce que fait le workflow (en une phrase)

Après que Claude Code (`/ingest`) a compilé `raw/ → wiki/`, ce workflow lit le manifeste `_archive_queue.json`, **archive chaque fichier brut original dans Google Drive**, puis **le supprime de `raw/`** — mais **seulement après confirmation de l'upload**. Résultat : `wiki/` reste la seule source qui fait foi, `raw/` redevient vide, et le brut est conservé dans Drive comme filet de sécurité (provenance).

**Principe directeur : l'IA étiquette, n8n exécute.** Claude Code sait dans quelle catégorie il a rangé chaque fichier et l'écrit dans le manifeste ; n8n se contente du déplacement physique.

---

## 2. Place dans le pipeline Lumina

```
Drive Inbox → (intake n8n) → GitHub raw/ → Claude Code /ingest compile → wiki/ + _archive_queue.json
   → push (Obsidian Git) → cross-trigger pgvector + Notion
                          → [CE WORKFLOW] vide raw/ vers Drive
```

Ce workflow est le **maillon manquant** : il consomme `_archive_queue.json` pour vider `raw/`.

---

## 3. Décisions de design (actées en début de build)

| Question | Décision | Pourquoi |
|---|---|---|
| Déclencheur | **Schedule Trigger** (cron `0 15 * * * *`) | Simple, robuste, zéro risque de boucle liée aux pushs. La minute `:15` désynchronise des autres workflows qui partent à `:00`. |
| Nettoyage du manifeste | **Non** (on s'appuie sur l'idempotence) | Si le fichier n'existe plus dans `raw/`, le Node 4 le skip (404). Les entrées périmées sont inoffensives. Moins de nodes. |
| Arborescence Drive | **v1 : à plat dans `Archive-Raw`** ; catégories en **v2** | Insérer un « search dossier » au milieu du flux fait perdre la pièce jointe binaire. On a d'abord validé le pipeline complet (surtout le garde-fou de suppression), les catégories viendront ensuite. |

---

## 4. Chaîne de nodes (v1 finale)

```
Schedule Trigger
   → Get a file (manifeste)
   → Code (décoder + parser le manifeste)
   → Get raw file (idempotence + contenu)   ──Success──→ Google Drive : Upload ──Success──→ GitHub : Delete a file
                                             └─Error (404 = skip)                └─Error (skip)
```

### Node 1 — Schedule Trigger
- **Quoi** : point de départ. Réveille le workflow tout seul.
- **Réglage** : Trigger Interval = **Custom (Cron)**, Expression = **`0 15 * * * *`**.
- **Comment lire le cron** : n8n utilise ici **6 champs** → `[seconde] [minute] [heure] [jour du mois] [mois] [jour de semaine]`. Donc `0 15 * * * *` = seconde 0, minute 15, toutes les heures, tous les jours.
- **Pourquoi `:15`** : décaler de `:00` pour ne pas charger le processeur en même temps que les autres workflows.

### Node 2 — Get a file (lire `_archive_queue.json`)
- **Quoi** : lit le **manifeste** = la liste de courses (quels fichiers archiver, dans quelle catégorie).
- **Réglages** : Credential GitHub ; Resource **File** / Operation **Get** ; Owner `frescatzi` ; Repo `ai-automation` ; File Path `_archive_queue.json` ; **As Binary = OFF** ; Reference `main`.
- **Pourquoi As Binary OFF** : on veut lire du **texte JSON**, pas un fichier binaire. (Au Node 4 ce sera l'inverse.)
- **Sortie** : l'API GitHub renvoie le contenu dans le champ `content`, **encodé en base64** (`encoding: "base64"`).

### Node 3 — Code (décoder + parser le manifeste)
- **Quoi** : transforme « 1 gros fichier texte » en **N items** (1 par fichier à archiver), pour traiter chacun individuellement.
- **Mode** : `Run Once for All Items`, JavaScript.
- **Code** :
  ```javascript
  const file = $input.first().json;
  const decoded = Buffer.from(file.content, 'base64').toString('utf-8');
  const entries = JSON.parse(decoded);
  return entries.map(e => ({ json: e }));
  ```
- **Pourquoi décoder** : GitHub emballe le contenu en base64 (alphabet « sûr » pour le transport JSON). `Buffer.from(..., 'base64')` le déballe, puis `JSON.parse` lit le tableau.
- **Sortie** : N items, chacun `{ path, category, processed_at }`.

### Node 4 — Get raw file (idempotence + contenu) ⭐
- **Quoi** : pour chaque entrée, récupère le fichier dans `raw/`. **Le récupérer prouve qu'il existe** → on fusionne « vérifier l'existence » et « récupérer le contenu » en un seul node.
- **Réglages** : GitHub Get a file ; File Path = `{{ $json.path }}` ; **As Binary = ON** (propriété `data`) ; **Settings → On Error = `Continue (using error output)`**.
- **Le cœur de l'idempotence** :
  - **Success** → le fichier existe → on a son contenu binaire prêt pour Drive.
  - **Error (404)** → déjà archivé lors d'un run précédent → **skip** (sortie error laissée dans le vide).
  - Relancer le workflow 10× : les fichiers déjà traités tombent en 404 et sont ignorés. Zéro doublon, zéro plantage.
- **Pourquoi As Binary ON ici** (vs OFF au Node 2) : on **transporte un fichier** vers Drive ; le binaire est le format adapté et marche pour tout type (md, image, pdf…).

### Node 5 — Google Drive : Upload
- **Quoi** : dépose chaque fichier binaire dans `Archive-Raw` (le filet de sécurité).
- **Réglages** : Credential Google Drive ; Resource **File** / Operation **Upload** ; **Input Data Field Name = `data`** ; **File Name = `{{ $binary.data.fileName }}`** ; **Parent Folder = By ID → `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`** (= `Archive-Raw`) ; **On Error = `Continue (using error output)`**.
- **Pourquoi `$binary.data.fileName`** : on lit le nom directement dans les métadonnées de la pièce jointe (déjà `2026-06-…​.md`).

### Node 6 — GitHub : Delete a file (le garde-fou) 🔒
- **Quoi** : vide `raw/` — l'étape sensible.
- **Branchement** : connecté **uniquement à la sortie Success de l'Upload**. Donc un fichier n'est supprimé **que si son upload a réussi**. Un upload raté part sur Error → jamais supprimé → retenté au prochain run.
- **Réglages** : GitHub Delete ; Owner `frescatzi` ; Repo `ai-automation` ; File Path = `{{ $('Code (décoder + parser le manifeste)').item.json.path }}` ; Commit Message = `archive: suppression de {{ $('Code (décoder + parser le manifeste)').item.json.path }} (copié dans Drive)` ; **Branch `main`** ; On Error = `Continue (using error output)`.
- **Pourquoi le pairing** (`$('Code …').item.json.path`) : après l'Upload, `$json` contient les infos Drive (plus le `path`). On va donc rechercher le chemin `raw/` d'origine dans le node Code ; n8n garde le lien (pairing) entre chaque item et sa ligne de manifeste.

---

## 5. Les garde-fous (et où ils vivent)

1. **Supprimer seulement après confirmation** → le Delete est branché sur la sortie **Success** de l'Upload (pas avant). Zéro perte de données.
2. **N'archiver QUE les fichiers du manifeste** → le Node 4 ne traite que les `path` listés. Les fichiers `raw/` non listés (ex. `README`, le fichier `vault: personal`) sont laissés intacts. *Vérifié en test.*
3. **Archive ≠ Inbox** → `Archive-Raw` est hors du dossier « Lumina Inbox » surveillé par l'intake → pas de boucle infinie. (Ils sont frères sous `OBSIDIAN`, pas l'un dans l'autre.)
4. **La suppression dans `raw/` ne déclenche pas pgvector/Notion** → ces workflows filtrent sur `wiki/`. À reconfirmer si on les retouche.

---

## 6. Challenges rencontrés → solutions → leçons

| # | Challenge | Cause | Solution | Leçon |
|---|---|---|---|---|
| 1 | `/ingest` → **404** au Node 2 | `_archive_queue.json` pas encore créé/poussé sur `main` | Lancer `/ingest` puis push via Obsidian Git | Le manifeste est un **pré-requis** produit par Claude Code + push, pas par n8n. Amorcer le pipeline avant de tester. |
| 2 | `/ingest` → **Unknown command** | La commande n'était pas installée dans le repo cible | Créer `.claude/commands/ingest.md` (et `sync.md`) dans le repo, puis redémarrer Claude Code | Une slash-command Claude Code est **scoped-projet** : elle doit vivre dans `<repo>/.claude/commands/`. |
| 3 | `cd` échoue | `~` déjà = `/Users/karter` (chemin doublé) + espace « Lumina AI » non échappé | Un seul des deux + **guillemets** autour du chemin | `~` ≠ à recombiner avec `/Users/...` ; toujours guillemeter les chemins avec espaces. |
| 4 | La minute du trigger non garantie | `Hours Between Triggers` compte depuis l'activation | Passer en **Cron `0 15 * * * *`** | Pour caler une minute précise (et désync de `:00`), utiliser une expression cron. n8n = cron **6 champs** (secondes en tête). |
| 5 | Contenu illisible au Node 2 | GitHub renvoie le contenu en **base64** | `Buffer.from(content, 'base64').toString('utf-8')` au Node 3 | Toujours décoder le `content` de l'API GitHub avant `JSON.parse`. |
| 6 | Upload → **« no binary file 'data' found »** | Le node **Edit Fields (Set)** intercalé ne transmettait pas le binaire | Brancher l'Upload **directement** en aval du node qui produit le binaire ; lire le nom via `$binary.data.fileName` | Garder le chemin binaire **direct** ; le Set node casse le transport de pièce jointe dans cette version. |
| 7 | `category`/`path` absents après le fetch | Le binaire **remplace** `$json` | **Pairing** : `{{ $('Code …').item.json.x }}` | Après un node qui change l'item (binaire, upload), récupérer les champs d'origine par référence au node source. |
| 8 | Sous-dossiers par catégorie compliqués | Un « search dossier » au milieu du flux **perd le binaire** | **v1 à plat** dans `Archive-Raw`, catégories en v2 | « Make it work, then make it nice » : valider le pipeline + le garde-fou critique d'abord. |

---

## 7. Tests réalisés (preuves)

- **Upload** : 13/13 sur Success, tous avec `parents = [Archive-Raw]`. Fichiers visibles dans Drive.
- **Delete (garde-fou)** : 13 commits de suppression sur `main` (récupérables : Drive + historique Git). `raw/` vidé des 13 fichiers ; `README` et `marketing-psychology` (non listés) laissés intacts.
- **Idempotence** : re-run complet → **Node 4 : Success = 0 / Error = 13** → Upload et Delete reçoivent 0 item. Aucun doublon, aucun plantage.

---

## 8. Infos techniques

- **n8n** : v2.10.4 self-hosté. Node GitHub v1.1, node Google Drive v3.
- **Repo** : `frescatzi/ai-automation`, branche `main`. Credential `GitHub account`.
- **Drive** : credential `Google Drive account - Live`. Dossier archive `Archive-Raw` = **ID `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`** (sous `LUMINA VPS/LUMINA AI/OBSIDIAN`, frère de `Lumina Inbox`). Inbox à NE PAS utiliser : `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`.
- **Sécurité** : aucun token/PAT dans les conversations — les creds vivent dans n8n.

---

## 9. Reste à faire / horizon

- **v2 — rangement par catégorie** : `Archive-Raw/<category>/`. Approche pressentie : pré-créer les sous-dossiers + un node de correspondance `folderName → folderId` (avec fallback racine), ou branche search/create + merge.
- **Croissance du manifeste** : `_archive_queue.json` est append-only. Les entrées périmées sont re-vérifiées (404) à chaque run — inoffensif mais croissant. Nettoyage optionnel à envisager plus tard.
- **Activation** : workflow à passer en **Active** pour que le cron tourne.
