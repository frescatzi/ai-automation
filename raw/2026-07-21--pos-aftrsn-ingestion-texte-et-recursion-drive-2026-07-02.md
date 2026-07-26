---
type: raw
title: "POS-AFTRSN_Ingestion-texte-et-recursion-Drive_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_ingestion-texte-et-recursion-drive_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# AFTRSN-LUMINA — POS EXACT — Ingestion texte/markdown + récursion Drive (étape 4b) — 2026-07-02

Deux briques d'ingestion vers `<brand>_memory`, alimentant chunk → embed → insert paramétré.

## 0. Références

| Élément | Valeur |
|---|---|
| Ingestion texte/md | `LUMINA-MEMORY-INGEST-TEXT` id `ugINEyyBKRLcDpH2` |
| Récursion Drive | `AFTRSN-MEMORY-INGESTION/DRIVE-RECURSIVE` id `8h3AowpjR9IBaxYF` |
| Dossier bible AFTRSN | `1PhE8-0vnO087fCWQnSQ3tnK27aFtPJF5` |
| Credentials | OpenAI embed `fqQvWAsn0tBEhSGU`, Postgres `Ojp0aTYdIFfNsr4u`, Drive `OMoBlrCPPTuBaJJL` |
| Décision | **MD + PDF uniquement** (docx exclus) |

## 1. Ingestion texte/markdown (réutilisable)

`executeWorkflowTrigger {brand, title, content, collection, source_ref, knowledge_type}` → **Chunk** (Code, `size=3000, overlap=200`, lit `content||document`, propage `brand`) → **Embed** (OpenAI `text-embedding-3-small`, 1 appel/chunk) → **Build Insert** (aligne `$input.all()` avec `$('Chunk').all()` par index ; table = `<brand>_memory` dérivée sanitizée ; INSERT paramétré `$1..$7`, `md5($2)` = `content_hash`, `ON CONFLICT DO NOTHING`) → **Postgres Insert** (`queryReplacement = {{ $json.values }}`). Appelable par outil ou pinData.

## 2. Récursion Drive (PDF + markdown)

`Manual`/`executeWorkflowTrigger {brand, folderId}` → **Drive Search** (`resource='fileFolder'`, `filter.folderId`, `returnAll`) → **Keep PDF+MD** (Code : ne garde que `application/pdf` / `text/markdown` / `text/plain` → drop docx + sous-dossiers) → **Download** (`fileId={{ $json.id }}`, binaire `data`) → **By Type** (Switch sur `mimeType` : branche pdf / branche texte) → **Extract PDF** (`operation:pdf`) ou **Extract Text** (`operation:text`), les deux → **Set Document** (`content={{ $json.text }}`, `title/source_ref = $('Keep PDF+MD').item.json.name`, `collection='canon'`) → Chunk → Embed → Build Insert → Postgres Insert. Testé bout en bout sur le dossier bible.

## 3. Pièges figés

1. **n8n ne parse pas le .docx** nativement → convertir en `.md`/`.pdf` d'abord (ou n'ingérer que PDF+md). Décision projet : MD + PDF only.
2. **Pairing après httpRequest/Extract** : le node Embed remplace `$json` par la réponse OpenAI → récupérer les métadonnées via `$('Chunk').all()[i]` (alignement par index) ou `$('Keep PDF+MD').item`.
3. **Idempotence** : `md5(content)` + `ON CONFLICT DO NOTHING` → ré-ingérer un fichier inchangé = no-op (pas de doublon). Pas de `TRUNCATE` (préserve episodic/skills/insights de la banque).
4. **Récursion 1 niveau** : `in parents` ne descend pas dans les sous-dossiers → boucler sur les sous-dossiers pour une récursion profonde (amélioration future).
5. Tester par API : `POST /rest/workflows/:id/run` + `triggerToStartFrom:{name:'Manual'}`.

---
*POS EXACT — 2026-07-02, Claude (Cowork). Version générique : POS-GENERIQUE ingestion multi-format vers banque vectorielle.*
