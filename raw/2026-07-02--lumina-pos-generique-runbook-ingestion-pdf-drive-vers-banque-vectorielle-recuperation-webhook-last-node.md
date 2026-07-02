---
type: raw
title: "LUMINA — POS GENERIQUE — Runbook ingestion PDF Drive vers banque vectorielle + recuperation webhook last-node"
source_url: "drive:1A84JvLrCrFmXILrk2rz9iq2IdChJ1Jq-"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Runbook : ingestion d'un PDF Google Drive → banque vectorielle pgvector & récupération webhook (mode last-node)

**Procédure Opérationnelle Standardisée (réutilisable, indépendante de la marque).**
**Cible :** tout pipeline RAG n8n (OpenAI embeddings → pgvector) + source Google Drive.

> Référence d'exploitation. Détail pédagogique : voir la « Marche à suivre GÉNÉRIQUE » correspondante.

---

## 1. Récupération webhook robuste (référence)

Chaîne : `Webhook → Embed (OpenAI) → Build Search → <Search node>` (le **dernier node** renvoie le JSON).
- `Webhook` : `Respond` = **When Last Node Finishes** ; `Response Data` = **All Entries**.
- **Aucun** node `Respond to Webhook` (sinon « Unused Respond to Webhook node found »).
- Search node : **Always Output Data** ON (banque vide → `[{}]`, pas de crash).
- Nom de table résolu via **allowlist / registre** (jamais d'interpolation d'un nom arbitraire → anti-injection).

## 2. Ingestion d'un fichier Drive unique (référence)

`(Truncate si reload) → Google Drive Download → Extract from File → Set Document → Chunk → Embed → Build Insert (paramétré) → Postgres Insert → Verify`.

- `Download file` : `File` = **By ID** (Fixed) = id du fichier ; binaire = champ **`data`**.
- `Extract from File` : **Extract From PDF** (PDF) ou **Extract from text file** (`.md`/`.txt`) ; `Input Binary Field` = `data` ; texte → `$json.text`.
- `Set Document` : `document={{ $json.text }}`, `title`, `source="Drive"`, `source_ref=<nom fichier>`, `collection`, `knowledge_type`.
- `Build Insert` / `Postgres Insert` : **requêtes paramétrées** (`$1..$n`, `md5($2)` pour `content_hash`, `$k::vector`) ; jamais de SQL concaténé.

## 3. Procédure « préparer la source Drive »

1. **Lister l'arborescence** du dossier source.
2. Si racine = sous-dossiers uniquement → la requête plate `'<folderId>' in parents` ne renvoie pas les fichiers profonds.
3. Choisir : **par ID** (1 fichier, MVP) ou **récursif** (multi-fichiers, avancé).
4. Récupérer l'**id** du fichier cible (via connecteur Drive ou l'URL `…/file/d/<ID>/view`).

## 4. Loop multi-fichiers (avancé)

- `Google Drive Search` (Advanced Search, `q` = `'<folderId>' in parents and trashed = false and mimeType = 'application/pdf'`) → boucle → `Download (By ID = {{ $json.id }})` → `Extract` → `Set Document`.
- Types mixtes → router par `mimeType` (Switch) : branche PDF (`Extract From PDF`) + branche texte (`Extract from text file`) → merge → `Set Document`.

## 5. Vérifier (toujours)

- Count par `source_ref` (node Verify).
- **End-to-end** : appel réel du point d'entrée (MCP/webhook).

## 6. Garde-fous

- Pipelines **TRUNCATE-puis-reload** : un échec post-truncate vide la table → ne relancer que si prêt à reconstruire.
- Embeddings = **coût** : ne ré-ingérer que si nécessaire.
- Champs Expression n8n : auto-close `{{ }}` → nettoyer le ` }}` en trop ; vérifier le Result.
- Connexions : se fier à la **sortie réelle** (Executions), pas au visuel du canvas.
- Nommage : séparer infra partagée et items de marque.

## 7. Challenges & leçons (mémo)

- Webhook qui renvoie une sortie intermédiaire = chaîne coupée en aval.
- Récupération robuste = mode « last node » sans node Respond.
- Dossier Drive imbriqué = cibler par ID (MVP) avant de tenter le récursif.
- `Download (binary 'data')` → `Extract from File` (type adapté) → `$json.text`.
- PDF très design = peu de texte ; prévoir une source texte (`.md`/`.docx`) si le contenu doit être net.
- Valider end-to-end.
