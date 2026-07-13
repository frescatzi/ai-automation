---
type: index
title: "Index — ai-automation"
vault: ai-automation
updated: 2026-07-13
---

# Index - coffre `ai-automation`

> Carte de navigation du `wiki/`. **Maintenue par le LLM** à chaque ingest.

## Concepts (wiki/)

### API Claude
- [[concept-limites-api-claude]] — limites de débit (RPM/ITPM/OTPM) et de dépenses de l'API Claude · status: active
- [[concept-prompt-caching]] — impact du cache de prompt sur le débit ITPM effectif · status: active
- [[concept-gestion-erreurs-429]] — stratégie de retry sur les erreurs 429, backoff, en-têtes utiles · status: draft

### Automation & Intégrations
- [[concept-oauth2-automation]] — patron OAuth2 universel (3 étapes, 3 erreurs classiques) applicable à tout outil d'automation · status: draft
- [[concept-n8n-credentials]] — gestion des credentials n8n (API Key, OAuth2, rotation, sécurité) · status: draft
- [[concept-intake-source-git]] — pattern idempotent & blindé Write→Verify→Delete-if-confirmed pour drainer une source vers Git · status: active
- [[concept-archivage-n8n-idempotent]] — pattern post-compilation : manifeste → archive → suppression sécurisée (4 garde-fous, pièges) · status: active
- [[concept-pipeline-memoire-wiki-git]] — hub parallèle wiki Git → base vectorielle (agents) + outil de doc (humains) ; table LLM dans la chaîne · status: active
- [[concept-classification-workflows-n8n]] — standard de rangement n8n : dossiers numérotés 01→05, 4 axes de tags, nommage, onboarding multi-marques · status: draft
- [[concept-validation-auto-ingest]] — validation de bout en bout du runner auto-ingest (raw → wiki → Git) + du déclenchement auto (push → n8n) ; artefact de test · status: draft

### IA & Agents
- [[concept-hermes-agent]] — Hermes-Agent (ex-Hermes-Exec) : bras d'exécution apprenant de LUMINA OS, cookie-auth, skills < 0.35, distinct du Router/Maestro · status: draft
- [[concept-routeur-multi-llm]] — routeur multi-LLM via OpenRouter : table task_type → modèle, 3-node architecture, règles de routage · status: draft
- [[concept-memoire-vectorielle-multi-marques]] — frontière physique multi-marques en pgvector, memory_registry, provisioning · status: draft
- [[concept-memoire-vivante-agents]] — mémoire vivante : 3 types, primitives WRITE/READ/CONSOLIDATE, boucle procédurale, LoRA · status: draft
- [[concept-bibliotheque-skills-apprenante]] — skills réutilisables à maturité (supervised→autonomous), 2 stockages, boucle consulter/journaliser/graduer/créer · status: draft

### Mémoire & Connaissance
- [[concept-capture-connaissance-debrief]] — débrief post-projet → mémoire (`collection=insights`) : modèle deux temps, gate humain, idempotence double garde · status: draft

## Synthèses (wiki/)

### Automation
- [[synthese-oauth2-n8n-google]] — configurer OAuth2 Google dans n8n, erreurs 401/403 courantes · status: active

### Système Lumina
- [[synthese-lumina-systeme-reference]] — référence complète du système Lumina : 4 couches (intake, wiki, pgvector, Notion), décisions figées (status: active)
- [[synthese-lumina-ai-os]] — Lumina AI OS : stack multi-LLM/multi-agents/multi-marques, routeur, mémoire vivante, playbook v1 (status: draft)

## Architecture (wiki/architecture/)

- [[architecture/Instructions_Projet_ChatGPT_n8n_v2]] — AI OS : rôles des outils, stack, principes, gouvernance, topologie hub-and-spoke (status: active)
- [[architecture/brief-montage-vaults]] — structure des 3 vaults Obsidian/Git, routeur de tri, conventions de nommage (status: active)
- [[architecture/Architecture_Connaissance_Obsidian_Centric]] — architecture centrée Obsidian (session 2026-06-20)
- [[architecture/Claude_Review_Knowledge_Governance_Layer_v1]] — revue gouvernance de la connaissance v1 (session 2026-06-20)
- [[architecture/Memoire_Centrale_ASP_Brief_Construction]] — brief construction mémoire centrale ASP (session 2026-06-20)
- [[architecture/Plan_Demarrage_Memoire_Centrale_MVP]] — plan de démarrage mémoire centrale MVP (session 2026-06-20)

## SOPs (wiki/sop/)

### Infrastructure pgvector & agents
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — installer pgvector sur Postgres Coolify, HNSW index, pièges (status: active)
- [[sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]] — système multi-agents hub-and-spoke + mémoire centrale + serveur MCP (status: active)

### Lumina (spécifique AFTRSN)
- [[sop/sop-lumina-intake-et-publish]] — robot intake Drive→GitHub + publish Notion idempotent, tous les pièges résolus (status: active)

### Générique (réutilisable)
- [[sop/sop-generique-pipeline-source-vers-vues]] — patron d'ingestion inbox→store + publication idempotente upsert-par-clé (status: active)

### Lumina — Pipeline d'archivage & mémoire
- [[sop/sop-lumina-archive-raw-vers-drive]] — workflow LUMINA Archive Raw→Drive : build, garde-fous, exploitation, incidents (status: active)
- [[sop/sop-creer-memoire-agents-humains]] — SOP Lumina mémoire agents+humains : pgvector + Notion + Obsidian Git, 4 parties + erreurs (status: active)
- [[sop/sop-lumina-auto-ingest-raw-vers-wiki]] — runner Claude headless (Coolify) déclenché par webhook, multi-coffres : automatise raw/ → wiki/ · status: draft
- [[sop/sop-generique-runner-llm-headless-webhook]] — patron générique réutilisable : orchestrateur no-code + runner LLM agentique headless · status: draft

### n8n & connexions API
- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer les clés API (Anthropic/OpenAI/Gemini), auto-héberger n8n, faire dialoguer les agents (status: active)
- [[sop/n8n-Brancher-API-et-Premier-Workflow]] — brancher les 3 credentials dans n8n.aftersunpeople.com, premier workflow Claude→ChatGPT→Gemini (status: active)

### Agents & orchestration
- [[sop/sop-cablage-orchestrateur-subagents]] — câbler Maestro → sub-agents + routeur LLM dans n8n, pièges, prompt orchestrateur (status: draft)
- [[sop/sop-audit-edition-n8n-api-interne]] — auditer/éditer n8n via API interne /rest (sans clé API), endpoints, template JS (status: draft)
- [[sop/sop-clonage-roster-agents]] — cloner un roster d'agents pour une nouvelle marque : 3 transforms, procédure complète (status: draft)
- [[sop/sop-agent-n8n-cookie-auth]] — brancher un agent n8n sur un service web avec auth cookie, Custom Auth credential (status: draft)
- [[sop/sop-apprendre-skill-a-hermes]] — écrire un skill directement dans la bibliothèque d'Hermès, contrat WRITE, test de trouvabilité (status: draft)

### Mémoire vectorielle
- [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] — diagnostiquer/réparer un pipeline RAG n8n+pgvector, SQL paramétré (status: draft)
- [[sop/sop-reparer-webhook-n8n-ingestion-pdf]] — réparer un webhook last-node + ingérer un PDF Drive dans une banque vectorielle (status: draft)
- [[sop/sop-ingestion-multi-format-banque-vectorielle]] — ingestion multi-format texte/PDF/dossier récursif vers pgvector (status: draft)

### Automation & contenus
- [[sop/sop-outreach-backfill]] — backfill de contacts manuels dans un pipeline d'outreach automatisé (status: draft)
- [[sop/sop-calendrier-contenu-agent]] — générateur de calendrier de contenu via agent → base, draft-only (status: draft)

## À compléter (trous repérés au lint)

_(rien encore)_

<!-- TEST Sync -->
