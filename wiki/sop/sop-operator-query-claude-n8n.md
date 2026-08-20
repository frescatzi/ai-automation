---
type: wiki
title: "SOP — Operator OS : canal query-claude (webhook n8n → API Anthropic)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-exact-operator-os-canal-query-claude-n8n-2026-08-17.md
related: ["sop-interroger-llm-externe-n8n-webhook", "concept-hermes-agent", "sop-dream-engine-cron-prescriptions-llm", "sop-routeur-multi-agent-operator-os", "concept-limites-api-claude", "concept-n8n-credentials", "synthese-lumina-ai-os"]
updated: 2026-08-17
---

# SOP — Operator OS : canal query-claude (webhook n8n → API Anthropic)

**Idée centrale :** implémentation de référence, chez **Operator OS/LUMINA**, du patron générique [[sop-interroger-llm-externe-n8n-webhook]] : le workflow n8n **`Operator-Query-Claude`** permet à Hermes/Operator OS d'interroger le modèle Claude (API Anthropic) avec des données agrégées de la stack, sans jamais exposer le secret Anthropic hors de n8n. ✅ Verrouillé, validé live au 2026-08-17.

## 1. Règle d'interrogation

- Workflow n8n `Operator-Query-Claude` : **Webhook POST `/query-claude`** → **HTTP Request** Anthropic `/v1/messages`.
- Modèle : `claude-sonnet-4-6`. Headers : `anthropic-version: 2023-06-01`, `content-type: application/json`. Auth : credential n8n `anthropicApi` (« Anthropic account », id `2CkrotK34KR7IVQF`) — cf. [[concept-n8n-credentials]].
- Entrée : `{prompt}`. Sortie : réponse texte de Claude. Le secret n'est jamais en clair dans le code.

## 2. Cas d'usage réels

- **[[sop-dream-engine-cron-prescriptions-llm]]** : Claude reçoit les agrégats (banques mémoire, skills, usage par profil) et produit 4 prescriptions JSON classées sévérité × dollar.
- **[[sop-routeur-multi-agent-operator-os]]** : le backend `claude-api` du routeur multi-agent appelle ce même webhook.
- Synthèse ad hoc : `scripts/ask_claude_stats.py` interroge Claude sur les données du dashboard.

## 3. Piège documenté

- Le modèle Claude n'a **PAS** accès à l'historique ni à la mémoire de Claude Cowork. C'est un LLM : il raisonne sur le contexte qu'on lui fournit en prompt, il n'exfiltre rien.
- Le canal interroge le **modèle**, pas l'historique. Pour l'historique, la source est l'export claude.ai (filtré) ou la mémoire pgvector.

## Difficultés / Solutions / Lessons learned

### Difficultés
- Crédit Anthropic épuisé → « Your credit balance is too low » (cf. [[concept-limites-api-claude]]).
- Le credential n8n ne résout pas dans `headerParameters` manuel → il faut passer par `authentication: predefinedCredentialType` + `nodeCredentialType: anthropicApi`.
- Premier test avec une vieille clé de brouillon à sec, alors que le credential actif était ailleurs.

### Solutions
- Recharger le crédit Anthropic, puis tester avec le credential n8n (jamais une clé en dur).
- Garder la clé dans le credential n8n, jamais dans le script ni le repo.

### Lessons learned
- Toujours tester l'endpoint Anthropic en `curl` direct d'abord pour isoler crédit vs câblage.
- Le bon modèle dépend du rôle (rédaction → Claude ; orchestration → OpenAI).

## Voir aussi

- [[sop-interroger-llm-externe-n8n-webhook]] — patron générique hors marque dont cette SOP est l'implémentation de référence.
- [[sop-dream-engine-cron-prescriptions-llm]] — consommateur principal de ce canal (brief quotidien de prescriptions).
- [[sop-routeur-multi-agent-operator-os]] — le backend `claude-api` du routeur multi-agent utilise ce même webhook.
- [[concept-hermes-agent]] — Hermes/Operator OS est l'appelant de ce canal.
- [[concept-limites-api-claude]] — comprendre l'erreur « credit balance too low ».
- [[concept-n8n-credentials]] — gestion du credential Anthropic utilisé ici.
- [[synthese-lumina-ai-os]] — place d'Operator OS dans l'écosystème LUMINA plus large.
