---
type: index
title: "Index — ai-automation"
vault: ai-automation
updated: 2026-06-29
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

## Synthèses (wiki/)

### Automation
- [[synthese-oauth2-n8n-google]] — configurer OAuth2 Google dans n8n, erreurs 401/403 courantes · status: active

### Système Lumina
- [[synthese-lumina-systeme-reference]] — référence complète du système Lumina : 4 couches (intake, wiki, pgvector, Notion), décisions figées (status: active)

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

### n8n & connexions API
- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer les clés API (Anthropic/OpenAI/Gemini), auto-héberger n8n, faire dialoguer les agents (status: active)
- [[sop/n8n-Brancher-API-et-Premier-Workflow]] — brancher les 3 credentials dans n8n.aftersunpeople.com, premier workflow Claude→ChatGPT→Gemini (status: active)

## À compléter (trous repérés au lint)

_(rien encore)_

<!-- TEST Sync -->
