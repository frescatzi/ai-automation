---
type: wiki
title: "AFTRSN — Workflow LUMINA-AI-Router (routage multi-LLM par task_type via OpenRouter)"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [AFTRSN/raw/2026-07-21--aftrsn-lumina-pos-exact-lumina-ai-router-routage-multi-llm-via-openrouter-2026-07-01.md]
related: [[methode-construction-workflow-n8n]], [[architecture-processus-metier]]
updated: 2026-07-21
---

# AFTRSN — Workflow LUMINA-AI-Router (routage multi-LLM par task_type via OpenRouter)

Sous-workflow n8n (`n8n.aftersunpeople.com`, id `Cu8SozYmondKM8RB`) servant de point d'entrée unique pour les appels LLM des agents Lumina (Maestro, spécialistes). Il choisit le modèle en fonction d'un `task_type` puis appelle OpenRouter — une passerelle unique OpenAI-compatible vers Claude/GPT/Gemini avec une seule clé, plutôt que N branches et 3 credentials séparés.

## Chaîne

`When Executed by Another Workflow (Accept all data)` → `Code (JavaScript)` → `HTTP Request`.

- Le trigger accepte tout le payload JSON (contrat : `routing.task_type`, `brand_profile`, `task_payload`).
- Le node Code mappe `routing.task_type` vers un slug OpenRouter et sort `{ model, task_type, messages:[{system},{user}] }`.
- Le node HTTP Request `POST https://openrouter.ai/api/v1/chat/completions` (credential type OpenRouter dédié, jamais de clé en clair dans un node) avec un body Expression `{{ JSON.stringify({ model: $json.model, messages: $json.messages }) }}`.

## Table de routage (bloc `M` du node Code)

| Rôle | task_types | Modèle (alias anti-churn) |
|---|---|---|
| Claude — rédaction/raisonnement | copy, reasoning, analysis, orchestrate | `~anthropic/claude-sonnet-latest` |
| GPT — idéation / défaut | draft (+ défaut) | `~openai/gpt-mini-latest` |
| GPT — image | image_gen | `~openai/gpt-latest` |
| Gemini Flash — recherche/volume | research, summarize_long, translate, multimodal, verify, classify, extract, preprocess, support_l1, realtime | `~google/gemini-flash-latest` |
| PII | private | **stop, aucun appel cloud** |

Les alias `~…-latest` sont des slugs flottants OpenRouter qui se mettent à jour seuls (anti-churn) — le `~` fait partie de l'id. Fallback pinnés si un alias casse : `anthropic/claude-opus-4.8`, `openai/gpt-5.5`, `google/gemini-3.5-flash`. Jamais de Haiku (retiré de l'API OpenRouter à date + bugs constatés). `openrouter/auto` est l'option à utiliser pour rendre une tâche insensible au churn de slugs.

## Garde-fou PII

`task_type = 'private'` interrompt le node Code avant tout appel réseau et renvoie `{error}` — règle non négociable : aucune donnée privée ne part vers le cloud. Un moteur local (Hermes, LoRA) est prévu pour absorber ce cas mais reste différé (pas de GPU disponible).

## Modifier la table de routage

1. Ouvrir le node Code → bloc `M`.
2. Ajouter/changer une ligne `task_type: 'provider/slug'` (vérifier le slug sur `openrouter.ai/models`, sujet à churn à chaque génération de modèle).
3. Pour un nouveau `task_type` avec logique spéciale (ex. PII), ajouter le garde-fou avant le mapping plutôt que dans la table.
4. Save → Publish (obligatoire pour que le changement soit pris en compte par les appelants).

## Coût et credentials

OpenRouter est un service tiers payant (compte + crédits, modèles `:free` disponibles pour tester gratuitement). La clé vit uniquement dans le credential n8n "OpenRouter account", jamais en dur dans un node.

## Tester

Le vrai test consiste à appeler le router depuis un workflow appelant réel (ex. Maestro) avec un payload conforme au contrat, puis vérifier le `model` sorti par le Code et la réponse OpenRouter (`choices[0].message.content`). Le test standalone (pin data sur le sub-trigger) est possible mais l'UI n8n est capricieuse sur ce point — préférer un appelant réel.

## Dépannage

- `model not found` → slug périmé, corriger dans le bloc `M` via `openrouter.ai/models`.
- `401` → credential OpenRouter (clé ou crédits épuisés).
- Sortie = objet requête au lieu d'une réponse → connexion Code → HTTP Request rompue, à reconnecter.
- `private` renvoie `{error}` : comportement normal, pas un bug (protection PII).

## Dette connue

- Câbler Maestro/agents pour appeler réellement le router (le test de routage en conditions réelles reste à faire).
- Système-prompts par rôle, au lieu du fallback générique « You are a helpful assistant ».
- Fallback/retry (`openrouter/auto`) et validation croisée pour les livrables critiques.
- Brancher la mémoire (RAG pgvector) dans `task_payload.context_raw` avant l'appel LLM.
- Moteur local pour le cas `private`/PII, conditionné à la disponibilité d'un GPU.
