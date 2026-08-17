---
type: raw
title: "POS-EXACT-routeur-multi-agent"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact-routeur-multi-agent.md"
captured: 2026-08-17
vault: ai-automation
brand: null
immutable: true
---

# POS-EXACT - Routeur multi-agent Operator OS (DeepSeek / Hermes / Claude)

**Date :** 2026-08-17 · **Projet :** Operator OS (LUMINA) · **Objet :** verrouiller le routeur qui permet de dialoguer avec 4 agents (DeepSeek, Hermes, Claude API, Claude Code) depuis une seule UI.
**Statut :** ✅ VERROUILLÉ & VALIDÉ LIVE.

---

## 1. Agents et backends

- **DeepSeek** (`deepseek/deepseek-v4-pro`) : backend `hermes-api` → API Hermes locale `http://127.0.0.1:8642/p/default/v1/chat/completions`.
- **Hermes** : backend `hermes-api` → même API, modèle `hermes`.
- **Claude API** (`claude-sonnet-4-6`) : backend `claude-api` → webhook n8n `query-claude` (workflow `Operator-Query-Claude`) → API Anthropic `/v1/messages`.
- **Claude Code** (`claude-2.1.220`) : backend `claude-code` → bridge Mac via tunnel Cloudflare.

## 2. Config exacte

| Élément | Valeur |
|---|---|
| Backend routeur | `/opt/data/operator-os/app/api_agents.py` |
| UI | `/opt/data/operator-os/app/static/index.html` (vue "Chat") |
| Routes API | `GET /api/agents`, `POST /api/agents/chat` |
| Définition des agents | `config.json` → clé `agents` |
| Auth | Basic Auth `OPERATOR_OS_USER` / `OPERATOR_OS_PASSWORD` |
| Port serveur | 8090 |

**Principe :** chaque agent porte un `type` qui détermine son backend. `hermes-api` appelle l'API locale, `claude-api` appelle n8n, `claude-code` appelle le bridge Mac. L'UI envoie `{agent_id, prompt}` et reçoit une réponse normalisée.

---

## Difficultés / Solutions / Lessons learned

### Difficultés
- Crédit Anthropic épuisé (`credit balance too low`) : le backend `claude-api` répondait en erreur.
- Le webhook n8n `query-claude` renvoie le raw Anthropic (tableau JSON), pas `{message}` : le parsing échouait.
- Le middleware BasicAuth bloquait les appels fetch du frontend.

### Solutions
- Recharger le crédit Anthropic (fait le 2026-08-17).
- Parser la réponse Anthropic (tableau `content[].text`) dans `_claude_api_chat`.
- Autoriser `/`, `/setup`, `/static/*` sans auth dans le middleware, et envoyer le header `Authorization` depuis le frontend.

### Lessons learned
- "Interroger Claude" via l'API = génération de texte, pas lecture de mémoire. La mémoire durable est dans pgvector.
- Le crédit API est un goulot d'étranglement : le documenter comme dépendance.
- Tester chaque backend en curl AVANT de brancher l'UI, pour isoler les erreurs.

---

*POS-EXACT routeur-multi-agent - 2026-08-17, verrouillé. Le config.json fait foi.*