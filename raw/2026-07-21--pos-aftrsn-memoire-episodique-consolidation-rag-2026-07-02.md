---
type: raw
title: "POS-AFTRSN_Memoire-episodique-consolidation-RAG_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_memoire-episodique-consolidation-rag_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# AFTRSN-LUMINA — POS EXACT — Mémoire épisodique + consolidation nocturne + RAG Router — 2026-07-02

Le système de mémoire « vivante » : les agents écrivent ce qu'ils font, une consolidation nocturne en tire des enseignements, et le Router injecte la mémoire pertinente avant chaque appel LLM créatif.

## 0. Références

| Élément | Valeur |
|---|---|
| Écriture | `LUMINA-MEMORY-WRITE/WEBHOOK` id `zu4jfZbmDz8trQLl` (executeWorkflowTrigger) |
| Récupération | `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` id `ZhUeKCo8nMp35gk1` (webhook `memory-search`) |
| Consolidation | `LUMINA-MEMORY-CONSOLIDATION/NIGHTLY` id `dVZyCwGYqnqMpS3P` (cron `0 3 * * *`) |
| Router | `LUMINA-AI-Router` id `Cu8SozYmondKM8RB` |
| Tables | `aftrsn_memory` / `lumina_memory` — colonnes `title, content, collection, knowledge_type, source, source_ref, content_hash (md5), embedding (1536)` |
| Collections | `canon` (bible), `episodic` (épisodes bruts), `insights` (consolidés), `docs` |

## 1. Écriture épisodique

- Sous-workflow `LUMINA-MEMORY-WRITE` : `executeWorkflowTrigger {brand,title,content,collection,knowledge_type,source,source_ref}` → Embed (OpenAI `text-embedding-3-small`) → Build Insert (INSERT paramétré, `md5(content)` = idempotence) → Postgres → Result.
- **⚠️ Pourquoi pas un webhook** : sur ce n8n en mode queue, les webhooks créés par API ne s'enregistrent pas (500 avant exécution). On utilise donc un `executeWorkflowTrigger`, appelé comme outil.
- **Maestro** a l'outil `Call 'Save Episode'` (→ ce sous-workflow, `brand=aftrsn`, `collection=episodic`, title/summary/topic via `$fromAI`). Ligne de prompt MEMORY : « après avoir traité une requête, appelle Save Episode une fois avec un résumé 2-3 phrases (demande, décision, résultat) ». Testé : Maestro décide un thème → épisode écrit.

## 2. Filtrage par collection (récupération)

- `memory-search` (Build Search) accepte `body.collection` : si fourni → `WHERE collection = '<coll>'` (sanitizé `[a-z0-9_]`) ; sinon → `WHERE collection IS DISTINCT FROM 'episodic'` (les épisodes bruts ne polluent pas les recherches canon). `limit` paramétrable (défaut 5, max 20). Renvoie aussi `collection`.

## 3. Consolidation nocturne

- Cron `0 3 * * *` (+ trigger Manual pour tester) → `SELECT title,content FROM aftrsn_memory WHERE collection='episodic' ORDER BY id DESC LIMIT 50` → Build Prompt → **Consolidate (LLM)** (OpenRouter `~anthropic/claude-sonnet-latest`, condense en 3-5 enseignements) → Extract Digest → Embed → INSERT `collection='insights'`, titre `Insights <date>`, `source='nightly-consolidation'`. **Garde tout** (pas de purge). Testé : insight écrit + récupérable via `collection=insights`.

## 4. RAG dans le Router

- Code node du Router : pour `task_type ∈ {copy, reasoning, analysis, draft}` et si instruction/context présent → `httpRequest` POST `memory-search {brand, question, limit:4}` → préfixe la mémoire trouvée dans `context_raw` (« MÉMOIRE DE MARQUE (contexte récupéré…) »). Best-effort (try/catch : si indispo, on continue sans). PII (`task_type='private'`) reste bloqué en amont. Testé : tâche `copy` sans contexte → voix de marque injectée.

## 5. Pièges figés

1. Webhooks créés par API **ne s'enregistrent pas** en mode queue → utiliser `executeWorkflowTrigger` (appel par outil) ou créer le webhook via l'UI.
2. Ne pas laisser `episodic` polluer les recherches canon → filtrage collection (exclusion par défaut).
3. Insight = 1 ligne/nuit (titre daté) pour éviter l'explosion ; `md5(content)` évite les doublons exacts. Consolidation actuelle = `ORDER BY id DESC LIMIT 50` (pas de `created_at` en base → incrémental impossible pour l'instant).
4. Embedding figé `text-embedding-3-small` 1536 — même modèle partout (écriture, recherche, insights) sinon distances incohérentes.
5. Tester par API : `POST /rest/workflows/:id/run` + `triggerToStartFrom` + `pinData` (le chat de l'éditeur est inutilisable en automation).

---
*POS EXACT — 2026-07-02, Claude (Cowork). Voir : POS-GENERIQUE mémoire vivante agents ; POS routeur multi-LLM ; POS Hermes outil Ops.*
