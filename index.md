# Index : ai-automation

> Carte de navigation du coffre. Toutes les pages sont reliées ici (aucune orpheline).

## Architecture

- [[Architecture_Connaissance_Obsidian_Centric]] : Architecture de la Connaissance · Modèle Git-hub (Obsidian + LLM Wiki)
- [[Claude_Review_Knowledge_Governance_Layer_v1]] : Analyse critique · Knowledge Governance Layer v1
- [[Instructions_Projet_ChatGPT_n8n_v2]] : Projet ChatGPT · Idéation, Architecture & Pré-analyse de l'écosystème IA / n8n
- [[Memoire_Centrale_ASP_Brief_Construction]] : Mémoire Centrale ASP · Brief de construction (MVP)
- [[Plan_Demarrage_Memoire_Centrale_MVP]] : Plan de démarrage · Mémoire Centrale (MVP)
- [[brief-montage-vaults]] : Architecture · Montage des 3 vaults Obsidian (brief de référence)

## Concepts

- [[concept-archivage-n8n-idempotent]] : Concept · Archivage idempotent post-compilation (pattern n8n)
- [[concept-bibliotheque-skills-apprenante]] : Concept · Bibliothèque de skills apprenante (maturité & graduation)
- [[concept-capture-connaissance-debrief]] : Concept · Capture de connaissance post-projet (débrief → mémoire)
- [[concept-classification-workflows-n8n]] : Standard de classification des workflows n8n (process-flow)
- [[concept-gestion-erreurs-429]] : Gestion des erreurs 429 · API Claude (et APIs REST)
- [[concept-hermes-agent]] : Concept · Hermes-Agent (bras d'exécution apprenant de LUMINA OS)
- [[concept-intake-source-git]] : Concept · Intake source → Git (pattern idempotent & blindé)
- [[concept-limites-api-claude]] : Limites de débit et de dépenses · API Claude
- [[concept-memoire-vectorielle-multi-marques]] : Mémoire vectorielle multi-marques (banque partagée + tables par marque)
- [[concept-memoire-vivante-agents]] : Mémoire vivante pour agents (épisodique + consolidation + RAG)
- [[concept-n8n-credentials]] : Credentials n8n · gestion et bonnes pratiques
- [[concept-oauth2-automation]] : OAuth2 · patron universel automation ↔ service cloud
- [[concept-pipeline-memoire-wiki-git]] : Concept · Pipeline mémoire depuis wiki Git (hub parallèle agents + humains)
- [[concept-prompt-caching]] : Prompt caching · impact sur le débit effectif (API Claude)
- [[concept-routeur-multi-llm]] : Routeur multi-LLM par task_type (passerelle OpenAI-compatible)
- [[concept-validation-auto-ingest]] : Concept · Validation du runner auto-ingest (raw → wiki → Git)

## Synthèses

- [[synthese-lumina-ai-os]] : LUMINA AI OS · Système multi-agents & multi-LLM
- [[synthese-lumina-systeme-reference]] : Lumina · Système de connaissance (référence complète)
- [[synthese-oauth2-n8n-google]] : Configurer OAuth2 Google dans n8n

## Procédures (SOP)

- [[Guide-Connexion-Agents-AI-n8n]] : Guide · Connecter Claude, ChatGPT et Gemini dans n8n
- [[sop-bridge-claude-code-mac-vps]] : SOP · Bridge Claude Code Mac↔VPS via tunnel Cloudflare (Operator OS)
- [[sop-dream-engine-cron-prescriptions-llm]] : SOP · Dream Engine, cron quotidien de prescriptions LLM (sévérité × impact $)
- [[SOP_installer-pgvector-sur-postgres-coolify]] : SOP · Installer pgvector sur un Postgres géré par Coolify
- [[SOP_systeme-multi-agents-memoire-centrale-mcp-n8n]] : SOP · Système multi-agents avec mémoire centrale & serveur MCP (n8n)
- [[n8n-Brancher-API-et-Premier-Workflow]] : n8n · Brancher les 3 API et créer le premier workflow multi-agents
- [[sop-agent-n8n-cookie-auth]] : SOP · Brancher un agent n8n sur un service web avec auth par cookie
- [[sop-apprendre-skill-a-hermes]] : SOP · Apprendre un skill à Hermès (écriture directe dans la bibliothèque)
- [[sop-audit-edition-n8n-api-interne]] : SOP · Auditer/éditer un n8n self-hosté par son API interne (sans clé API)
- [[sop-bridge-cli-local-tunnel-serveur-distant]] : SOP générique · Relier un agent serveur à un CLI IA local via tunnel
- [[sop-cablage-orchestrateur-subagents]] : SOP · Câbler un orchestrateur AI Agent à des sub-agents et un routeur LLM (n8n)
- [[sop-calendrier-contenu-agent]] : SOP · Générateur de calendrier de contenu (agent → base, draft-only)
- [[sop-clonage-roster-agents]] : SOP · Clonage d'un roster d'agents vers une nouvelle marque
- [[sop-creer-memoire-agents-humains]] : SOP Lumina · Créer la mémoire (agents + humains) depuis le wiki Git
- [[sop-diagnostiquer-pipeline-memoire-vectorielle]] : SOP · Diagnostiquer/réparer un pipeline de mémoire vectorielle (n8n + pgvector)
- [[sop-generique-pipeline-source-vers-vues]] : SOP générique · Pipeline d'ingestion & publication idempotente (n8n)
- [[sop-generique-runner-llm-headless-webhook]] : SOP générique · Runner LLM agentique headless déclenché par webhook
- [[sop-ingestion-multi-format-banque-vectorielle]] : SOP · Ingestion multi-format (texte/markdown + PDF/dossier) vers banque vectorielle
- [[sop-lumina-archive-raw-vers-drive]] : SOP Lumina · Archive Raw→Drive (n8n)
- [[sop-lumina-auto-ingest-raw-vers-wiki]] : SOP Lumina · Auto-Ingest raw→wiki (runner Claude headless, multi-coffres)
- [[sop-lumina-intake-et-publish]] : SOP Lumina · Intake (Drive→GitHub) & Publication Notion idempotente
- [[sop-outreach-backfill]] : SOP · Backfill : intégrer des contacts déjà démarchés manuellement dans un outreach automatisé
- [[sop-reparer-credential-postgres-partagee-n8n]] : SOP · Réparer une credential Postgres partagée fantôme (n8n / mémoire agents)
- [[sop-reparer-webhook-n8n-ingestion-pdf]] : SOP · Réparer un webhook n8n (last-node) + ingérer un PDF Drive dans une banque vectorielle
- [[sop-repondeur-email-drafts-agent]] : SOP · Répondeur email à brouillons via agent (pattern draft-only)
- [[sop-synchroniser-sqlite-deux-conteneurs-docker]] : SOP générique · Synchroniser une base SQLite entre deux conteneurs Docker (docker cp + cron)
- [[sop-token-systemuser-meta-ads-n8n]] : SOP · Token System User Meta Ads pour n8n (création de campagnes)
