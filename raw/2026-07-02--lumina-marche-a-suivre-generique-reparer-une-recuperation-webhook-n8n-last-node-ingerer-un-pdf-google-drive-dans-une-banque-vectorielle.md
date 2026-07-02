---
type: raw
title: "LUMINA — Marche a suivre GENERIQUE — Reparer une recuperation webhook n8n (last-node) + ingerer un PDF Google Drive dans une banque vectorielle"
source_url: "drive:1p6tJKgfp7pTXHMkCk6kd4Ue9X9FsZh6P"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre GÉNÉRIQUE — Réparer une récupération webhook n8n (mode last-node) & ingérer un PDF Google Drive dans une banque vectorielle pgvector

**Portée :** patron réutilisable, indépendant de la marque. S'applique à tout pipeline RAG sur n8n (embeddings OpenAI → pgvector) exposé par webhook + MCP, et à toute ingestion d'un document Drive (PDF) vers une table vectorielle.

> Convention : explications en français ; noms de nodes/credentials et code en anglais. Numérotation à partir de 1.

---

## A. Réparer une récupération (webhook → search) qui renvoie n'importe quoi

### 1. Qualifier précisément la panne
- `NodeApiError` ⇒ un node interne plante.
- Réponse = **la sortie d'un node intermédiaire** (ex. une requête SQL construite, pas les résultats) ⇒ **la chaîne est coupée en aval** de ce node.
- « No Respond to Webhook node found » ⇒ le mode `responseNode` est actif mais le flux n'atteint jamais un node Respond valide (souvent : connexion rompue plus haut).

### 2. Trouver le point de coupure
1. n8n → **Executions** → ouvrir l'exécution rouge → lire la sortie réelle.
2. Comparer la sortie attendue (résultats) vs réelle (sortie intermédiaire) : le **dernier node ayant produit la sortie** est le bout vivant de la chaîne ; le lien vers le node suivant est rompu.
3. **Ne pas se fier au visuel** du canvas : une connexion peut paraître présente mais être rompue à l'exécution.

### 3. Reconnecter + simplifier (récupération robuste)
- Recréer à la main la connexion rompue (drag sortie → entrée).
- Préférer un webhook **sans dépendance à un node `Respond to Webhook`** :
  - Node `Webhook` : `Respond` = **When Last Node Finishes**, `Response Data` = **All Entries**.
  - **Supprimer** tout node `Respond to Webhook` résiduel (même désactivé il déclenche « Unused Respond to Webhook node found »).
  - Le **dernier node** de la chaîne (ex. `Postgres Search`) renvoie directement le JSON.
- **Publish**, puis valider par l'appel réel (MCP/webhook).

## B. Ingérer un PDF Google Drive vers une banque vectorielle

### 4. Inspecter l'arborescence Drive AVANT de coder
- Lister le dossier source. **Si la racine ne contient que des sous-dossiers**, une requête plate `'<folderId>' in parents` ne renverra **pas** les fichiers profonds.
- Décider : **fichier unique par ID** (MVP simple, fiable) **ou** traversée récursive (puissant mais lourd). Pour démarrer, le fichier unique par ID est recommandé.

### 5. Chaîne d'ingestion d'un fichier unique
`(Truncate si reload complet) → Google Drive Download → Extract from File → Set Document → Chunk → Embed (OpenAI) → Build Insert (paramétré) → Postgres Insert → Verify`.

- **`Google Drive` → `Download file`** : `File` = **By ID** (mode Fixed) avec l'id du fichier ; sortie binaire = champ **`data`**.
- **`Extract from File`** : opération selon le type (**Extract From PDF** pour un PDF ; **Extract from text file** pour `.md`/`.txt`). `Input Binary Field` = **`data`**. Texte extrait dans `$json.text`.
- **`Set Document`** (Manual Mapping) — mapper les métadonnées de la banque :
  - `document` = `{{ $json.text }}`
  - `title` = libellé lisible du document
  - `source` = `Drive`
  - `source_ref` = nom de fichier (traçabilité)
  - `collection` = catégorie (ex. `canon`, `voice`, `ops`)
  - `knowledge_type` = ex. `standard`
- **Embed / Build Insert / Postgres Insert** : voir le patron « requêtes paramétrées » (jamais de SQL concaténé).

### 6. Loop sur plusieurs fichiers (étape avancée)
- Pour ingérer plusieurs PDF d'une arborescence imbriquée : `Google Drive Search` (par sous-dossier) → boucle → `Download (By ID = {{ $json.id }})` → `Extract` → `Set Document`.
- Types mixtes (PDF **et** md) : router par `mimeType` (Switch) → branche `Extract From PDF` / branche `Extract from text file` → merger avant `Set Document`. Un node `Extract` = **un seul type**.

### 7. Valider end-to-end
- Exécuter ; vérifier le **count** par `source_ref`.
- Re-tester par l'**appel MCP réel** de la banque (`search_brand_memory(brand=...)`).

## C. Challenges typiques & leçons

- **Réponse = sortie intermédiaire** → connexion coupée en aval ; recréer le lien (ne pas se fier au visuel).
- **Webhook robuste** → mode « last node » + **aucun** node Respond résiduel.
- **Dossier Drive imbriqué** → la requête plate `in parents` ne voit que les sous-dossiers ; inspecter d'abord, puis cibler par ID (MVP) ou récursif.
- **Drive → texte** → `Download (binary 'data')` → `Extract from File` (type adapté) → `$json.text`.
- **Champs Expression n8n** → auto-close `{{ }}` ; nettoyer le ` }}` en trop (End + 3 Backspace) ; vérifier le **Result**.
- **PDF design** → peu de texte extractible ; prévoir un `.md`/`.docx` converti si le contenu doit être net.
- **Validation** → toujours end-to-end (appel MCP), pas node par node.
- **Nommage** → séparer infra partagée (préfixe infra) et items de marque (préfixe marque).
