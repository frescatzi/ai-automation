---
type: raw
title: "AFTRSN-LUMINA — POS EXACT — Runbook ingestion bible de marque (Drive PDF vers aftrsn_memory) et recuperation simplifiee"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — pos exact — runbook ingestion bible de marque (drive pdf vers aftrsn_memory) et recuperation simplifiee.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# POS EXACT — Runbook : ingestion de la bible de marque (Drive PDF → `aftrsn_memory`) & récupération simplifiée

**Procédure Opérationnelle Standardisée (spécifique à ce système).**
**Cible :** pgvector (DB `n8n`) · n8n `n8n.aftersunpeople.com` · MCP `LUMINA-MCP-Server`.

> Référence d'exploitation. Détail pédagogique : voir « Marche à suivre EXACTE — … 2026-06-30 ».

---

## 1. État de référence (au 2026-06-30)

- **`lumina_memory`** — banque partagée (automation/process), ~66 lignes (ex-`asp_memory` → `aios_memory` → `lumina_memory`).
- **`aftrsn_memory`** — banque de marque AFTER SUN PEOPLE ; contient la bible (`collection='canon'`, 2 chunks au 2026-06-30).
- **`memory_registry`** — routage : `lumina → lumina_memory`, `aftrsn → aftrsn_memory`.
- Embeddings : OpenAI `text-embedding-3-small` (credential `OpenAi account`). DB : `Postgres account`. Drive : `Google Drive account - Live`.

## 2. Workflows clés

| Workflow | id | Rôle | État |
|---|---|---|---|
| `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` | `ZhUeKCo8nMp35gk1` | Récupération (webhook `memory-search`) | Published |
| `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR` | `OUWDFF1HUhBr0Vy1` | Ingestion banque partagée → `lumina_memory` | Published |
| `AFTRSN-MEMORY-INGESTION/DRIVE→AFTRSN_MEMORY` | `krWBy2Sj9k47H24z` | Ingestion bible → `aftrsn_memory` | Published |

## 3. Récupération — câblage de référence (simplifié)

`Webhook → Embed Question (OpenAI) → Build Search → Postgres Search`.
- `Webhook` : `Respond` = **When Last Node Finishes**, `Response Data` = **All Entries**. **Pas** de node `Respond to Webhook`.
- `Build Search` : allowlist `{ lumina:'lumina_memory', aftrsn:'aftrsn_memory' }`, défaut `lumina` ; construit la requête `SELECT id, title, left(content,200) AS extrait, round((1-(embedding <=> $vec))…) AS similarite FROM <table> ORDER BY embedding <=> $vec::vector LIMIT 5`.
- `Postgres Search` : **Always Output Data** ON (banque vide → `[{}]`, pas de crash).

⚠️ Ne **jamais** remettre un node `Respond to Webhook` avec ce mode (« Unused Respond… »).

## 4. Ingérer / mettre à jour la bible (`aftrsn_memory`)

Workflow `AFTRSN-MEMORY-INGESTION/DRIVE→AFTRSN_MEMORY`. Chaîne :
`Truncate aftrsn_memory → Google Drive Download (By ID) → Extract From PDF → Set Document → Chunk → Embed → Build Insert → Postgres Insert`.

- **Fichier ciblé (MVP) :** `20260215_Corporate Brand Identity.pdf`, file id **`1nJwBnEA0a7fDlSG2-NgjCagIupzn8a2e`** (Drive : `… / AFTRSN_Official Documents / Brand Business`).
- **`Download file`** : `File` = By ID (Fixed) ; binaire = `data`.
- **`Extract from File`** : Extract From PDF ; `Input Binary Field` = `data` ; texte → `$json.text`.
- **`Set Document`** : `document={{ $json.text }}`, `title="Corporate Brand Identity"`, `source="Drive"`, `source_ref="20260215_Corporate Brand Identity.pdf"`, `collection="canon"`, `knowledge_type="standard"`.
- **Lancer** : `Execute workflow` → vérifier les nodes verts (Chunk/Embed/Insert = N items) → **Publish**.

⚠️ `Truncate aftrsn_memory` au début = **reload complet** de la marque. Un échec après le truncate vide la banque.

## 5. Vérifier

- Count : node `Verify Table Count` (`… FROM aftrsn_memory GROUP BY source_ref`).
- End-to-end : MCP `search_brand_memory(brand=aftrsn, question="…")` → doit renvoyer le contenu de la bible.
- Contrôle santé partagée : MCP `search_brand_memory(brand=lumina)` → 5 lignes.

## 6. Dette / à faire connue

- [ ] **Étape 4b** : récursion sur l'arborescence Drive (autres PDF) + branche markdown (`Extract from text file`).
- [ ] Profondeur du texte : le PDF design ne donne que 2 chunks → ingérer un `.md`/`.docx` converti pour capter les genres (Afro House / Afro Tech / Melodic Afro House).
- [ ] Produire des `.md` propres dans Obsidian `brands/aftrsn`.
- [ ] Brancher les 3 spécialistes sur `brand=aftrsn` (Superviseur reste `lumina`).
- [ ] (Optionnel) filtrage `collection` dans la recherche du Gardien-marque ; durcir l'auth MCP (none → OAuth).

## 7. Challenges & leçons (mémo)

- Webhook qui renvoie une sortie intermédiaire → connexion coupée en aval (recréer le lien).
- Récupération robuste = « last node finishes » + aucun node Respond.
- Dossier Drive imbriqué → cibler par ID (MVP) plutôt que requête plate `in parents`.
- `Download (binary 'data')` → `Extract From PDF` → `$json.text`.
- Champs Expression : auto-close `{{ }}` (nettoyer le ` }}` en trop).
- Pipelines TRUNCATE-reload : un échec post-truncate vide la table.
- Valider end-to-end (appel MCP).
