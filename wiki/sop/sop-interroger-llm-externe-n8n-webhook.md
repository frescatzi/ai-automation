---
type: wiki
title: "SOP générique — Interroger un LLM externe depuis n8n (webhook → HTTP Request → credential)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-generique-interroger-llm-externe-n8n-webhook-2026-08-17.md
related: ["concept-n8n-credentials", "sop-operator-query-claude-n8n", "concept-limites-api-claude", "sop-generique-runner-llm-headless-webhook"]
updated: 2026-08-17
---

# SOP générique — Interroger un LLM externe depuis n8n (webhook → HTTP Request → credential)

**Idée centrale :** patron réutilisable, hors marque, pour tout appel **stateless** à une API de LLM externe (Anthropic, OpenAI, etc.) depuis n8n. Un workflow expose un webhook POST qui reçoit un prompt, appelle l'API du LLM via un node HTTP Request, et renvoie la réponse. Le secret vit **uniquement dans un credential n8n** — jamais en clair dans un header écrit à la main. Instance concrète de ce patron : [[sop-operator-query-claude-n8n]] (Operator OS/LUMINA → API Anthropic).

> Ne pas confondre avec [[sop-generique-runner-llm-headless-webhook]] : ce dernier documente un **runner LLM agentique** (lit un dépôt, décide, écrit, committe) trop riche pour un simple appel API. Ici, on parle d'un appel **synchrone prompt→texte**, sans autonomie ni écriture dans une source de vérité.

## Marche à suivre

1. Créer un workflow : **Webhook** (POST, `responseMode: lastNode`) → **HTTP Request** vers l'endpoint du LLM.
2. HTTP Request : `authentication: predefinedCredentialType`, `nodeCredentialType` = celui du provider (ex. `anthropicApi`).
3. Headers fixes : version d'API + `content-type`. Body JSON construit en expression (`JSON.stringify`).
4. Tester le webhook en `curl`. Si l'erreur est « credit balance too low », le câblage est bon mais le **crédit** est épuisé (cf. [[concept-limites-api-claude]]).

## Garde-fous

1. Ne jamais mettre la clé en dur dans `headerParameters` manuel si un credential existe : utiliser `predefinedCredentialType` (cf. [[concept-n8n-credentials]]).
2. Toujours tester l'endpoint en `curl` direct d'abord pour isoler crédit vs câblage.
3. Ne pas confondre « interroger le modèle » avec « lire l'historique » : un LLM raisonne sur le contexte fourni en prompt, il n'a pas de mémoire propre — pour l'historique, la source est un export ou une mémoire dédiée (pgvector).

## Difficultés rencontrées & solutions

| Difficulté | Solution |
|---|---|
| « Error in workflow » opaque via l'API n8n | Tester l'API en `curl` direct avec la clé pour isoler la cause réelle |
| Credential qui ne résout pas dans `headerParameters` manuel | Basculer sur `authentication: predefinedCredentialType` + `nodeCredentialType` |

## Leçons clés

- Le crédit épuisé se déguise en erreur de workflow opaque — toujours tester l'API en direct avant de suspecter le câblage n8n.
- Un LLM n'est pas une base de données : il synthétise un contexte qu'on lui fournit, il n'exfiltre pas d'historique.

## Voir aussi

- [[sop-operator-query-claude-n8n]] — implémentation concrète de ce patron chez Operator OS (webhook `/query-claude` → API Anthropic).
- [[concept-n8n-credentials]] — patron général de gestion des secrets/clés (`predefinedCredentialType`, jamais en dur).
- [[concept-limites-api-claude]] — comprendre l'erreur « credit balance too low » et les limites de débit/dépense API Claude.
- [[sop-generique-runner-llm-headless-webhook]] — patron voisin mais distinct : runner LLM agentique headless (autonomie + écriture), pas un simple appel stateless.
