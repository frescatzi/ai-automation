---
type: wiki
title: "SOP générique — Registre d'agents multi-backends derrière une interface unique"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-generique-routeur-multi-agent.md
related: ["concept-routeur-multi-llm", "sop-cablage-orchestrateur-subagents", "sop-bridge-cli-local-tunnel-serveur-distant", "concept-n8n-credentials"]
updated: 2026-08-17
---

# SOP générique — Registre d'agents multi-backends derrière une interface unique

**Idée centrale :** patron réutilisable, hors marque, pour toute UI/dashboard qui doit dialoguer avec **plusieurs LLM hétérogènes** (modèle interne, API cloud, CLI local) depuis un seul écran. On définit un **registre d'agents** — chaque agent a un `type` qui détermine son backend — et un routeur qui lit ce `type` pour aiguiller vers le bon canal. L'UI ne connaît que deux contrats génériques (`{agent_id, prompt}` en entrée, `{response}` en sortie) et jamais les secrets des backends.

Complémentaire à [[concept-routeur-multi-llm]] : ce dernier route **par `task_type`** (nature de la tâche) à travers une passerelle unique (OpenRouter) ; ce patron route **par `agent_id`** (identité de l'agent choisi dans l'UI) vers des backends de nature différente (API interne, webhook cloud, CLI local via bridge). Les deux couches peuvent se composer : un agent du registre peut lui-même appeler le routeur multi-LLM en interne.

## Marche à suivre

1. Définir un registre d'agents (fichier de config) : `id`, `name`, `type`, `model`, `description`, `status`.
2. Implémenter un backend par `type` : API interne, API cloud via webhook, CLI local via bridge/tunnel (cf. [[sop-bridge-cli-local-tunnel-serveur-distant]]).
3. Exposer deux routes : `GET /agents` (liste du registre) et `POST /agents/chat` (`{agent_id, prompt}`).
4. Dans l'UI, afficher une carte par agent avec un indicateur de statut (online/offline) et une zone de chat.
5. Router selon le `type` et renvoyer une réponse normalisée (`{response}` unique, quel que soit le backend).

## Garde-fous

1. Ne jamais exposer un secret dans l'UI : le backend gère les clés (cf. [[concept-n8n-credentials]] pour le patron général de gestion des secrets).
2. Tester chaque backend en CLI avant de le brancher à l'UI.
3. Normaliser la réponse (`response` unique) quel que soit le backend, pour que l'UI n'ait à gérer qu'un seul format.
4. Garder un statut par agent (online/offline) pour signaler tout backend indisponible avant que l'utilisateur n'envoie un prompt.

## Difficultés rencontrées & solutions

| Difficulté | Solution |
|---|---|
| Un backend répond dans un format brut (tableau JSON) différent du format attendu | Parser le format brut dans la fonction du backend (extraire le texte des `content[]`) |
| Un middleware d'auth bloque les appels du frontend | Autoriser les routes statiques sans auth et envoyer l'en-tête d'auth depuis le frontend |
| Un crédit API épuisé rend un backend silencieux | Afficher le statut dégradé et documenter la dépendance crédit |

## Leçons clés

- L'API d'un LLM génère du texte, elle ne lit pas une mémoire : séparer génération et mémoire durable.
- Isoler chaque backend derrière une fonction dédiée facilite le débogage.
- Le statut live d'un agent (vert/rouge) est plus parlant qu'un message d'erreur après coup.

## Voir aussi

- [[concept-routeur-multi-llm]] — routage complémentaire par `task_type` à travers une passerelle unique (OpenRouter) ; à composer, pas à confondre.
- [[sop-cablage-orchestrateur-subagents]] — variante n8n : un orchestrateur AI Agent délègue à des sub-agents et au routeur LLM via `toolWorkflow`.
- [[sop-bridge-cli-local-tunnel-serveur-distant]] — patron pour le backend `type: cli local` quand le CLI IA tourne derrière NAT.
- [[concept-n8n-credentials]] — gestion des secrets/clés que le backend ne doit jamais exposer à l'UI.
