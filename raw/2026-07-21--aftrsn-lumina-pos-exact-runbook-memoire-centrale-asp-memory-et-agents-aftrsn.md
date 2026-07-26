---
type: raw
title: "AFTRSN-LUMINA — POS EXACT — Runbook memoire centrale (asp_memory) et agents AFTRSN"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — pos exact — runbook memoire centrale (asp_memory) et agents aftrsn.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS EXACT — Runbook : mémoire centrale `asp_memory` & agents AFTRSN

**Procédure Opérationnelle Standardisée (spécifique à ce système).**
**Système :** `n8n.aftersunpeople.com` · pgvector `asp_memory` · MCP `LUMINA-MCP-Server`.
**Validateur final :** Katel.

> Checklist opérationnelle concise. Pour le détail pédagogique, voir la « Marche à suivre EXACTE » du 2026-06-29.

---

## 1. Cartographie du système (référence)

**Mémoire (infra LUMINA, dossier `LUMINA-02-MEMORY`)**
- `LUMINA-MCP-Server` — MCP Server Trigger, outil `search_brand_memory` → POST webhook `memory-search`. (Relais ; « réussit » même si l'aval plante.)
- `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` — Webhook → **Embed Question (OpenAI)** → Build Search → **Postgres Search** → Respond to Webhook.
- `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR` — **Truncate** → fetch GitHub → Chunk → **Embed (OpenAI)** → **Build Insert** → **Postgres Insert** → **Verify Table Count**.
- `LUMINA-MEMORY-INGESTION/PUBLISH-NOTION`, `LUMINA-RETRIVIAL-MEMORY (MANUAL/TEST)`, `LUMINA-dédup / idempotence`.
- Table `asp_memory` : `id, title, content, collection, knowledge_type, source, source_ref, content_hash, embedding VECTOR(1536)` ; index HNSW cosine ; embeddings OpenAI `text-embedding-3-small`.
- Credential embeddings/DB : **`OpenAi account`** / **`Postgres account`**.

**Agents (marque AFTRSN)**
- `AFTRSN-SUPERVISOR` (dossier AFTRSN-AGENTS) · 3 spécialistes dans `Sub-Agents` : `AFTRSN-BRAND-GUARDIAN`, `AFTRSN-CHANNEL-CONTENT`, `AFTRSN-ACQUISITION-PERFORMANCE`.
- Modèle agents : OpenAI **gpt-4.1-mini** (Responses API), credential `OpenAi account`.
- Outil mémoire des agents : node **`Knowledge Bank`** → webhook `memory-search`.

## 2. Si la mémoire ne répond plus (MCP en erreur)

1. n8n → **Executions** → repérer le workflow en **Error** (probable : `LUMINA-RETRIEVAL-MEMORY/WEBHOOK`).
2. Ouvrir l'exécution → node rouge → lire l'erreur.
3. **Si « credential … does not exist »** : ouvrir le node `Embed (OpenAI)` → re-sélectionner **OpenAi account** → Save. Vérifier le **même** node dans `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR`.
4. **Si 0 résultat / Invalid JSON** : la table est probablement vide → voir §3.
5. Re-tester via le MCP `search_brand_memory`.

## 3. Repeupler `asp_memory`

1. Ouvrir `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR`.
2. Vérifier que `Embed (OpenAI)` pointe sur **OpenAi account**.
3. Vérifier que `Postgres Insert` est en **paramétré** : Query = `{{ $json.query }}` ; Options → **Query Parameters** (Expression) = `{{ $json.values }}`.
4. **Execute workflow** → tous verts.
5. Exécuter `Verify Table Count` → attendu ~**25 fichiers / 66 chunks** (varie selon le wiki).
6. **Publish** avec note de version.
7. Test final : MCP `search_brand_memory`.

> ⚠️ L'ingestion fait un **TRUNCATE en premier**. Ne la lancer que si on est prêt à reconstruire toute la table. Un échec après le truncate laisse `asp_memory` vide.

## 4. Vérifier les agents

1. Rechercher `AFTRSN` → les 4 agents doivent être **Published** (ignorer `ZZZ_…-DEPRECATED`).
2. Superviseur : AI Agent + Chat Model (OpenAi account / gpt-4.1-mini) + Chat Memory + **Knowledge Bank** + 3 **Call-workflow** vers les spécialistes.
3. Spécialistes : 2 triggers (chat + Executed by Another Workflow) + Knowledge Bank ; prompt `{{ $json.query || $json.chatInput }}`.
4. Test rapide : taper un message dans le Chat du Superviseur (ex. « Quelle est la règle de nommage de la marque ? ») et vérifier qu'il interroge la mémoire / délègue.

## 5. Dette technique connue (à traiter)

- [ ] Aligner le nom de l'outil : node `Knowledge Bank` vs prompt « Knowledge ».
- [ ] Corriger la doc : modèle = **gpt-4.1-mini** (pas gpt-4o-mini).
- [ ] Ajouter aux prompts spécialistes : « respond in the user's language » + garde-fou « Katel valide le critique ».
- [ ] Spécialistes : confirmer le fallback `sessionId` de la Chat Memory en sous-workflow (`{{ $json.sessionId || 'subagent' }}`).
- [ ] Enrichir `asp_memory` avec le contenu de **marque** (bible) — actuellement surtout du wiki système.
- [ ] Durcir l'auth du MCP (OAuth2) — actuellement « no auth » (MVP).

## 6. Challenges & leçons (mémo)

- Erreur opaque → Executions → node rouge.
- Credential affichée OK mais binding mort → réassigner ; balayer retrieval + ingestion.
- TRUNCATE-puis-reload → un échec vide la table.
- SQL concaténé casse sur markdown → **paramètres** (`$1..$n` + Array).
- Éditeur d'expression n8n auto-ferme `{{ }}` → nettoyer.
- Valider end-to-end (appel MCP).
