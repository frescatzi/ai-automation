---
type: wiki
title: "SOP — Routeur multi-agent Operator OS (DeepSeek / Hermes / Claude API / Claude Code)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-exact-routeur-multi-agent.md
related: ["sop-registre-agents-multi-backends", "sop-bridge-claude-code-mac-vps", "sop-operator-query-claude-n8n", "concept-hermes-agent", "concept-limites-api-claude", "synthese-lumina-ai-os"]
updated: 2026-08-17
---

# SOP — Routeur multi-agent Operator OS (DeepSeek / Hermes / Claude API / Claude Code)

**Idée centrale :** implémentation de référence, chez **Operator OS (LUMINA)**, du patron générique [[sop-registre-agents-multi-backends]] : un routeur permet de dialoguer avec **4 agents hétérogènes** (DeepSeek, Hermes, Claude API, Claude Code) depuis une seule UI. ✅ Verrouillé, validé live au 2026-08-17. Le `config.json` fait foi.

## 1. Agents et backends

- **DeepSeek** (`deepseek/deepseek-v4-pro`) : backend `hermes-api` → API Hermes locale `http://127.0.0.1:8642/p/default/v1/chat/completions`.
- **Hermes** : backend `hermes-api` → même API, modèle `hermes`.
- **Claude API** (`claude-sonnet-4-6`) : backend `claude-api` → webhook n8n `query-claude` (workflow `Operator-Query-Claude`, cf. [[sop-operator-query-claude-n8n]]) → API Anthropic `/v1/messages`.
- **Claude Code** (`claude-2.1.220`) : backend `claude-code` → bridge Mac via tunnel Cloudflare (cf. [[sop-bridge-claude-code-mac-vps]]).

## 2. Config exacte

| Élément | Valeur |
|---|---|
| Backend routeur | `/opt/data/operator-os/app/api_agents.py` |
| UI | `/opt/data/operator-os/app/static/index.html` (vue « Chat ») |
| Routes API | `GET /api/agents`, `POST /api/agents/chat` |
| Définition des agents | `config.json` → clé `agents` |
| Auth | Basic Auth `OPERATOR_OS_USER` / `OPERATOR_OS_PASSWORD` |
| Port serveur | 8090 |

**Principe :** chaque agent porte un `type` qui détermine son backend. `hermes-api` appelle l'API locale, `claude-api` appelle n8n, `claude-code` appelle le bridge Mac. L'UI envoie `{agent_id, prompt}` et reçoit une réponse normalisée — application concrète du contrat générique décrit dans [[sop-registre-agents-multi-backends]].

## Difficultés / Solutions / Lessons learned

### Difficultés
- Crédit Anthropic épuisé (`credit balance too low`) : le backend `claude-api` répondait en erreur (cf. [[concept-limites-api-claude]]).
- Le webhook n8n `query-claude` renvoie le raw Anthropic (tableau JSON), pas `{message}` : le parsing échouait.
- Le middleware BasicAuth bloquait les appels `fetch` du frontend.

### Solutions
- Recharger le crédit Anthropic (fait le 2026-08-17).
- Parser la réponse Anthropic (tableau `content[].text`) dans `_claude_api_chat`.
- Autoriser `/`, `/setup`, `/static/*` sans auth dans le middleware, et envoyer le header `Authorization` depuis le frontend.

### Lessons learned
- « Interroger Claude » via l'API = génération de texte, pas lecture de mémoire. La mémoire durable est dans pgvector.
- Le crédit API est un goulot d'étranglement : le documenter comme dépendance.
- Tester chaque backend en `curl` AVANT de brancher l'UI, pour isoler les erreurs.

## Voir aussi

- [[sop-registre-agents-multi-backends]] — patron générique hors marque dont cette SOP est l'implémentation de référence.
- [[sop-bridge-claude-code-mac-vps]] — détail du backend `claude-code` (bridge Mac via tunnel Cloudflare).
- [[sop-operator-query-claude-n8n]] — détail du backend `claude-api` (webhook n8n → API Anthropic).
- [[concept-hermes-agent]] — Hermes est à la fois un backend du routeur (`hermes-api`) et l'infrastructure qui héberge une partie d'Operator OS.
- [[concept-limites-api-claude]] — comprendre l'erreur « credit balance too low » rencontrée sur le backend `claude-api`.
- [[synthese-lumina-ai-os]] — place d'Operator OS dans l'écosystème LUMINA plus large.
