# log.md — coffre `ai-automation`

> Append-only. Ingests, Q&A modifiant le wiki, lints. Format greppable.
> `## [AAAA-MM-JJ] <action> | <objet>` — actions : `ingest` · `qa` · `lint` · `schema-change`.

---

## [2026-06-20] schema-change | Création du coffre ai-automation (v2.2)
## [2026-06-22] ingest | Limites de débit/dépenses API Claude → wiki/concept-limites-api-claude.md, wiki/concept-prompt-caching.md
## [2026-06-22] ingest | SOP OAuth2 n8n + Google → wiki/synthese-oauth2-n8n-google.md
## [2026-06-22] lint   | raw/2026-06-22--marketing-psychologie-du-consommateur.md hors périmètre (vault: personal dans son propre frontmatter) — pas de page wiki/ créée ici, à déplacer vers le coffre personal
## [2026-06-24] ingest | Lumina système référence complète → wiki/synthese-lumina-systeme-reference.md (status: active, publish: notion)
## [2026-06-24] ingest | SOP Lumina intake Drive→GitHub + publish Notion idempotent → wiki/sop/sop-lumina-intake-et-publish.md
## [2026-06-24] ingest | SOP générique pipeline ingestion & publication idempotente → wiki/sop/sop-generique-pipeline-source-vers-vues.md
## [2026-06-24] ingest | SOP système multi-agents + MCP (existant) → frontmatter + related ajoutés à wiki/sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md
## [2026-06-24] ingest | SOP installer pgvector Coolify (existant) → frontmatter + related ajoutés à wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
## [2026-06-24] ingest | Guide connexion agents AI n8n (existant) → frontmatter + related ajoutés à wiki/sop/Guide-Connexion-Agents-AI-n8n.md
## [2026-06-24] ingest | n8n brancher API & premier workflow (existant) → frontmatter + related ajoutés à wiki/sop/n8n-Brancher-API-et-Premier-Workflow.md
## [2026-06-24] ingest | Instructions projet ChatGPT n8n v2 (existant) → frontmatter + related ajoutés à wiki/architecture/Instructions_Projet_ChatGPT_n8n_v2.md
## [2026-06-24] ingest | Brief montage vaults Obsidian → wiki/architecture/brief-montage-vaults.md
## [2026-06-24] ingest | Batch 2026-06-24 — 9 sources raw/ traitées, index.md mis à jour (Architecture + SOP sections ajoutées)
## [2026-06-24] lint   | Audit backlinks : synthese-oauth2-n8n-google (related vide → relié à Guide-Connexion-Agents-AI-n8n + sop-lumina-intake-et-publish) ; concept-limites-api-claude (related étendu aux 3 nouvelles pages) ; frontmatter ajouté à 4 pages architecture sans en-tête (Architecture_Connaissance_Obsidian_Centric, Claude_Review_Knowledge_Governance_Layer_v1, Memoire_Centrale_ASP_Brief_Construction, Plan_Demarrage_Memoire_Centrale_MVP)
## [2026-06-29] ingest | OAuth2 patron universel → wiki/concept-oauth2-automation.md (CRÉÉ, draft) · source : raw/2026-06-22--concept-configurer-oauth2-automation.md
## [2026-06-29] ingest | Limites API Claude enrichi (best-practices agents, note Lumina) → wiki/concept-limites-api-claude.md (MAJ) · source : raw/2026-06-23--concept-claude-api-rate-limits.md
## [2026-06-29] ingest | Stubs créés pour concepts référencés manquants : wiki/concept-gestion-erreurs-429.md · wiki/concept-n8n-credentials.md
## [2026-06-29] ingest | Backlinks ajoutés : synthese-oauth2-n8n-google → concept-oauth2-automation + concept-n8n-credentials ; concept-prompt-caching → concept-gestion-erreurs-429
## [2026-06-29] ingest | _archive_queue.json créé (13 entrées : 11 runs précédents + 2 de ce run) — exclut raw/2026-06-22--marketing-psychologie-du-consommateur.md (hors périmètre, vault: personal)
## [2026-06-29] ingest | Pull git : 12 nouvelles sources raw/ 2026-06-29 → batch ingest ci-dessous
## [2026-06-29] ingest | brief-claudecode-montage-vaults-obsidian (re-export) → sources + related mis à jour dans wiki/architecture/brief-montage-vaults.md
## [2026-06-29] ingest | guide-connexion-agents-ai-n8n (re-export, même contenu) → sources mis à jour dans wiki/sop/Guide-Connexion-Agents-AI-n8n.md
## [2026-06-29] ingest | sop-installer-pgvector-sur-postgres-coolify (re-export) → sources + related mis à jour dans wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
## [2026-06-29] ingest | sop-systeme-multi-agents-memoire-centrale-mcp-n8n (re-export) → sources + related mis à jour dans wiki/sop/SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md
## [2026-06-29] ingest | sop-intake-github-idempotent-blinde → sources + related mis à jour dans wiki/sop/sop-lumina-intake-et-publish.md ; concept-intake-source-git + sop-lumina-archive-raw-vers-drive ajoutés en related
## [2026-06-29] ingest | howto-generique-archiver + sop-generique-workflow-archivage → CRÉÉ wiki/concept-archivage-n8n-idempotent.md (status: active)
## [2026-06-29] ingest | howto-generique-intake-source-vers-git → CRÉÉ wiki/concept-intake-source-git.md (status: active)
## [2026-06-29] ingest | howto-generique-creer-la-memoire-agents-et-humains → CRÉÉ wiki/concept-pipeline-memoire-wiki-git.md (status: active)
## [2026-06-29] ingest | howto-archive-raw-vers-drive-lumina + sop-lumina-archive-raw-vers-drive → CRÉÉ wiki/sop/sop-lumina-archive-raw-vers-drive.md (status: active)
## [2026-06-29] ingest | sop-creer-la-memoire-agents-et-humains → CRÉÉ wiki/sop/sop-creer-memoire-agents-humains.md (status: active)
## [2026-06-29] ingest | index.md mis à jour : +3 concepts Automation, +2 SOPs Lumina pipeline archivage/mémoire ; _archive_queue.json étendu (+12 entrées)
## [2026-07-06] lint   | audit backlinks post-ingest — 2 wikilinks cassés corrigés (session-2026-06-25 + spec-obsidian pointaient vers raw/ au lieu de wiki/) · 3 orphelins reliés : sop-agent-n8n-cookie-auth (← sop-cablage-orchestrateur-subagents + synthese-lumina-ai-os) · sop-outreach-backfill et sop-calendrier-contenu-agent (← synthese-lumina-ai-os)
## [2026-07-06] ingest | /sync batch 2026-07-02/07-06 — git pull (53 fichiers) + PURGE (0/0, queue []) + ingest complet · CRÉÉ : wiki/synthese-lumina-ai-os.md · wiki/concept-routeur-multi-llm.md · wiki/concept-memoire-vectorielle-multi-marques.md · wiki/concept-memoire-vivante-agents.md · wiki/concept-classification-workflows-n8n.md · wiki/sop/sop-cablage-orchestrateur-subagents.md · wiki/sop/sop-audit-edition-n8n-api-interne.md · wiki/sop/sop-clonage-roster-agents.md · wiki/sop/sop-agent-n8n-cookie-auth.md · wiki/sop/sop-diagnostiquer-pipeline-memoire-vectorielle.md · wiki/sop/sop-reparer-webhook-n8n-ingestion-pdf.md · wiki/sop/sop-ingestion-multi-format-banque-vectorielle.md · wiki/sop/sop-outreach-backfill.md · wiki/sop/sop-calendrier-contenu-agent.md · MAJ : wiki/synthese-lumina-systeme-reference.md (section Lumina AI OS v2) · wiki/sop/sop-lumina-archive-raw-vers-drive.md (spec v2 sous-dossiers) · wiki/concept-pipeline-memoire-wiki-git.md (table LLM dans la chaîne + /ingest vs /sync) · index.md (+11 pages) · _archive_queue.json (51 entrées, 6 catégories)
