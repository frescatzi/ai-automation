---
type: raw
title: "POS-AFTRSN_Cablage-Maestro-subagents-Router_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_cablage-maestro-subagents-router_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# AFTRSN-LUMINA — POS EXACT — Câblage Maestro ↔ sub-agents + LUMINA-AI-Router — 2026-07-02

Procédure exacte du câblage réalisé le 02-07 (corrections A1, A2, A6). État final : chat → Maestro → délégation → routage multi-LLM → synthèse, tout en PROD.

## 0. Références

| Élément | Valeur |
|---|---|
| Maestro | `OlrQO21u178SjgBK` (10 nodes) |
| Culture-Steward | `vK6B2WuDfbse41Jv` |
| Experience-Designer | `bBu5ycRwHoK7ACdV` |
| Secretary (ex « Experience / Secrétaire ») | `FtPFOKvi2t18DFb1` |
| Marketing | `wyUEojwZjSKFRaRL` |
| Channel-Content (hors tools Maestro) | `MWUwLi3vu2Ws65lL` |
| LUMINA-AI-Router | `Cu8SozYmondKM8RB` |

## 1. A1 — Tool-nodes de Maestro (délégation)

1. Supprimer les 3 nodes obsolètes : Call Channel-Content, Call Acquisition-Performance, Call « Brand-Buardian » (typo → cible inexistante).
2. Créer 4 nodes `@n8n/n8n-nodes-langchain.toolWorkflow` (typeVersion 2.2), un par sub-agent :
   - `parameters.workflowId` = `{__rl:true, value:<id>, mode:'list', cachedResultName:<nom>}`
   - `parameters.workflowInputs` : mappingMode `defineBelow`, value `{query: "={{ $fromAI('query', '<description de la requête>') }}"}`, schema une colonne `query` (string)
   - `parameters.description` = contrat lisible par l'agent (quand déléguer à qui)
3. Connexions : `connections["<nom du node>"] = {ai_tool: [[{node:'AI Agent', type:'ai_tool', index:0}]]}`.
4. Publier. **Test** : demander dans le chat une délégation explicite à un expert → vérifier le node vert + verdict dans la réponse.

## 2. A2 — Réparer le Secrétaire

1. Coller le prompt v2 (§4 du doc *System-prompts 1ère vague v2 — 01-07*) dans `AI Agent → options.systemMessage`.
2. **Credential manquante** = cause de l'inactivation : copier la credential Postgres du node `memoryPostgresChat` d'un agent sain (Marketing) vers celui du Secrétaire.
3. Renommer le workflow → `AFTRSN-Secretary`. Publier/activer.

## 3. A6 — Maestro → Router

1. Ajouter sur Maestro un toolWorkflow → `Cu8SozYmondKM8RB` avec 3 entrées `$fromAI` : `task_type` (liste des types dans la description), `instruction`, `context_raw`.
2. Ajouter au prompt de Maestro une section `ROUTING:` (utiliser le Router pour les tâches lourdes/spécialisées ; jamais de PII).
3. **Côté Router** — deux adaptations :
   - Code node : accepter le contrat À PLAT en plus de l'imbriqué — `p.routing?.task_type || p.task_type` et `tp.context_raw || p.context_raw`, `tp.instruction || p.instruction`.
   - **Trigger `executeWorkflowTrigger` (mode jsonExample) : élargir l'exemple JSON** pour déclarer TOUS les champs acceptés (plat + imbriqué). Sans ça, les champs non déclarés sont droppés silencieusement → le Router reçoit les défauts (`draft`).
4. Publier les deux workflows. **Test** : « Router avec task_type research : … » → vérifier dans l'output du tool `model = google/gemini-3.5-flash-…`.

## 4. Validations du 02-07

- Alias OpenRouter `~…-latest` : **fonctionnels** (résolution live constatée). Mapping du Router inchangé.
- Fallbacks pinnés présents au catalogue : `anthropic/claude-opus-4.8`, `openai/gpt-5.5`, `google/gemini-3.5-flash`.

## 5. Dépannage

- Sub-workflow reçoit des valeurs par défaut → champs absents du jsonExample du trigger (§3.3).
- Publication refusée « Missing required credential » → credential absente sur un node (§2.2).
- Modifs API invisibles en PROD → PATCH ne fait qu'un draft ; publier via `POST /rest/workflows/:id/activate` (voir POS API interne).

---
*POS EXACT — 2026-07-02, Claude (Cowork). Voir aussi : POS-GENERIQUE câblage orchestrateur, POS API interne n8n, MILESTONE du 02-07.*
