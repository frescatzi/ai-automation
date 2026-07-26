---
type: wiki
title: "AFTRSN — Runbook mémoire centrale asp_memory & agents"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [AFTRSN/raw/2026-07-02--aftrsn-lumina-pos-exact-runbook-memoire-centrale-asp-memory-et-agents-aftrsn.md, AFTRSN/raw/2026-07-07--aftrsn-lumina-pos-exact-runbook-memoire-centrale-asp-memory-et-agents-aftrsn.md, AFTRSN/raw/2026-07-21--aftrsn-lumina-pos-exact-runbook-memoire-centrale-asp-memory-et-agents-aftrsn.md]
related: [[workflow-lumina-ai-router]]
updated: 2026-07-21
---

# AFTRSN — Runbook mémoire centrale `asp_memory` & agents

Procédure opérationnelle standardisée pour diagnostiquer et remettre en service la mémoire vectorielle centrale AFTRSN (`asp_memory`, Postgres/pgvector) et les agents qui s'y appuient. Système : `n8n.aftersunpeople.com` + MCP `LUMINA-MCP-Server`. Validateur final : Katel.

## Cartographie

**Mémoire** (infra LUMINA, dossier `LUMINA-02-MEMORY`) :
- `LUMINA-MCP-Server` : outil `search_brand_memory` → POST webhook `memory-search`. Relais — « réussit » même si l'aval plante, donc ne pas se fier à son statut seul.
- `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` : Webhook → Embed Question (OpenAI) → Build Search → Postgres Search → Respond to Webhook.
- `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR` : Truncate → fetch GitHub → Chunk → Embed (OpenAI) → Build Insert → Postgres Insert → Verify Table Count.
- Autres workflows liés : `LUMINA-MEMORY-INGESTION/PUBLISH-NOTION`, `LUMINA-RETRIVIAL-MEMORY (MANUAL/TEST)`, `LUMINA-dédup / idempotence`.
- Table `asp_memory` : `id, title, content, collection, knowledge_type, source, source_ref, content_hash, embedding VECTOR(1536)` ; index HNSW cosine ; embeddings OpenAI `text-embedding-3-small`.
- Credentials : **`OpenAi account`** (embeddings) / **`Postgres account`** (DB).

**Agents** (marque AFTRSN) :
- `AFTRSN-SUPERVISOR` (dossier `AFTRSN-AGENTS`) avec 3 spécialistes dans `Sub-Agents` : `AFTRSN-BRAND-GUARDIAN`, `AFTRSN-CHANNEL-CONTENT`, `AFTRSN-ACQUISITION-PERFORMANCE`.
- Modèle : OpenAI **gpt-4.1-mini** (Responses API), credential `OpenAi account`.
- Outil mémoire des agents : node **`Knowledge Bank`** → webhook `memory-search`.

## Si la mémoire ne répond plus (MCP en erreur)

1. n8n → Executions → repérer le workflow en Error (souvent `LUMINA-RETRIEVAL-MEMORY/WEBHOOK`).
2. Ouvrir l'exécution → node rouge → lire l'erreur.
3. Si « credential … does not exist » : rouvrir le node `Embed (OpenAI)` → re-sélectionner **OpenAi account** → Save. Vérifier le même node côté ingestion (`WIKI-GITHUB→PGVECTOR`).
4. Si 0 résultat / Invalid JSON : la table est probablement vide → voir la procédure de repeuplement ci-dessous.
5. Re-tester via le MCP `search_brand_memory`.

## Repeupler `asp_memory`

1. Ouvrir `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR`.
2. Vérifier que `Embed (OpenAI)` pointe sur **OpenAi account**.
3. Vérifier que `Postgres Insert` est en paramétré : Query = `{{ $json.query }}` ; Options → Query Parameters (Expression) = `{{ $json.values }}`.
4. Execute workflow → tous verts.
5. Exécuter `Verify Table Count` → attendu ~25 fichiers / 66 chunks (varie selon le wiki).
6. Publish avec note de version.
7. Test final : MCP `search_brand_memory`.

⚠️ L'ingestion fait un **TRUNCATE en premier**. Ne la lancer que si on est prêt à reconstruire toute la table — un échec après le truncate laisse `asp_memory` vide.

## Vérifier les agents

1. Rechercher `AFTRSN` → les 4 agents doivent être **Published** (ignorer `ZZZ_…-DEPRECATED`).
2. Superviseur : AI Agent + Chat Model (OpenAi account / gpt-4.1-mini) + Chat Memory + `Knowledge Bank` + 3 Call-workflow vers les spécialistes.
3. Spécialistes : 2 triggers (chat + Executed by Another Workflow) + `Knowledge Bank` ; prompt `{{ $json.query || $json.chatInput }}`.
4. Test rapide : message dans le Chat du Superviseur (ex. « Quelle est la règle de nommage de la marque ? ») et vérifier qu'il interroge la mémoire / délègue.

## Dette technique connue

- Aligner le nom de l'outil : node `Knowledge Bank` vs prompt « Knowledge ».
- Corriger la doc : le modèle est **gpt-4.1-mini** (pas gpt-4o-mini).
- Ajouter aux prompts spécialistes : « respond in the user's language » + garde-fou « Katel valide le critique ».
- Spécialistes : confirmer le fallback `sessionId` de la Chat Memory en sous-workflow (`{{ $json.sessionId || 'subagent' }}`).
- Enrichir `asp_memory` avec du contenu de **marque** (bible) — actuellement surtout du wiki système.
- Durcir l'auth du MCP (OAuth2) — actuellement « no auth » (MVP).

## Challenges & leçons

- Erreur opaque → toujours partir de Executions → node rouge.
- Une credential affichée OK peut avoir un binding mort → réassigner, balayer retrieval **et** ingestion.
- Le cycle TRUNCATE-puis-reload signifie qu'un échec en cours de reload vide la table.
- Le SQL concaténé casse sur du markdown → toujours utiliser des paramètres (`$1..$n` + Array).
- L'éditeur d'expression n8n auto-ferme les `{{ }}` → nettoyer après édition.
- Toujours valider end-to-end via un appel MCP réel, pas juste le statut des workflows.
