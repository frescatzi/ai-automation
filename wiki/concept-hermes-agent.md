---
type: wiki
title: "Concept — Hermes-Agent (bras d'exécution apprenant de LUMINA OS)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-07-12--canon-lumina-hermes-agent-2026-07-12.md"
related:
  - "concept-bibliotheque-skills-apprenante"
  - "concept-memoire-vivante-agents"
  - "concept-routeur-multi-llm"
  - "sop/sop-cablage-orchestrateur-subagents"
  - "sop/sop-agent-n8n-cookie-auth"
  - "synthese-lumina-ai-os"
  - "sop/sop-apprendre-skill-a-hermes"
updated: 2026-07-13
---

# Concept — Hermes-Agent (bras d'exécution apprenant de LUMINA OS)

> **Idée centrale :** Hermes-Agent est le composant de [[synthese-lumina-ai-os]] qui **fait réellement les tâches** (recherche web, opérations) — pendant que les agents **raisonnent et rédigent**, lui **agit** — et qui **apprend** en réutilisant les procédures déjà éprouvées ([[concept-bibliotheque-skills-apprenante]]).

## Alias & identité (fiche canon)

- **LUMINA-Hermes-Agent** = **ex-LUMINA-Hermes-Exec** = « Hermes Exec » = « Hermes Ops ». Toute mention de « Hermes-Exec » désigne ce même composant.
- Workflow n8n **`Dwv4rcMqNAyQzlrF`** — nom live **LUMINA-Hermes-Agent** — **actif** — renommé *Exec → Agent* le 2026-07-11.
- Error Workflow = **Sentinel** (`xzaH0uWy0idKVphF`).

## Rôle : exécuter + apprendre

- **Exécuter** les tâches d'opérations et de recherche que les agents *décident* mais ne *font* pas eux-mêmes (les agents raisonnent / rédigent ; Hermes agit).
- **Apprendre** : avant chaque tâche, il recherche par **similarité** les *skills* pertinents (procédures apprises, `collection = skills`) dans les banques `lumina` + `<marque>_memory`, **seuil de distance < 0.35**, et les injecte dans son contexte ; il **journalise** l'usage des skills (`Log Skills`) → boucle d'amélioration continue, **consolidée la nuit**. C'est l'application concrète de la [[concept-bibliotheque-skills-apprenante]] et de la mémoire procédurale décrite dans [[concept-memoire-vivante-agents]].

## Fonctionnement (le fil)

1. Reçoit une tâche `{ message }` — via `On Workflow Call` (appel par un autre workflow) ou `On Chat Message` (test).
2. Prépare l'entrée → **recherche skills** (Postgres/pgvector) → **exécute via le moteur Hermes** self-hosted (`hermes.aftersunpeople.com`, Coolify) en **cookie-auth** → **journalise** (`Log Skills`) → **renvoie le résultat**.
3. Le moteur Hermes répond actuellement via un **gateway modèle** (famille Claude Opus) ; cible future = **modèle local**.

## Qui l'appelle

- Les sous-agents **Secretary** et **Marketing**, via l'outil `Call 'Hermes (Ops)'`.
- Le bot **Lyra** : chemin **M4 Web/Hermes** (`LUMINA-TELEGRAM-WEB-HERMES`, `RRAZbO4zEb1sFNvb`) pour les intentions `web_search` / `hermes_ops`.

Le câblage « orchestrateur → sous-agents → outil Hermes » est détaillé dans [[sop/sop-cablage-orchestrateur-subagents]].

## Sécurité & contrat

- **Contrat d'entrée** : `{ message }`.
- **Auth moteur** : **cookie-auth** via credential **Custom Auth** — le mot de passe vit **uniquement dans la credential**, jamais dans un node, un prompt ou un log. Voir le patron réutilisable [[sop/sop-agent-n8n-cookie-auth]].

## Ce que Hermes n'est PAS (frontières)

| Composant | Responsabilité | À ne pas confondre |
|---|---|---|
| **Hermes-Agent** | **Exécution** + apprentissage (skills) | — |
| **Router** ([[concept-routeur-multi-llm]], `LUMINA-AI-Router`) | Choix du LLM par `task_type` | Hermes ne choisit pas le modèle |
| **Maestro** | Orchestration des sous-agents | Hermes n'orchestre pas |

## Voir aussi

- [[concept-bibliotheque-skills-apprenante]] — les *skills* qu'Hermes consulte et gradue.
- [[concept-memoire-vivante-agents]] — mémoire épisodique + procédurale dont Hermes est le principal consommateur/producteur.
- [[concept-routeur-multi-llm]] — composant frère (choix du modèle), distinct de l'exécution.
- [[sop/sop-cablage-orchestrateur-subagents]] — comment Maestro et les sous-agents appellent Hermes.
- [[sop/sop-agent-n8n-cookie-auth]] — le patron cookie-auth utilisé par le moteur Hermes.
- [[synthese-lumina-ai-os]] — place de Hermes dans le stack LUMINA AI OS.
- [[sop/sop-apprendre-skill-a-hermes]] — procédure vérifiée pour écrire un skill directement dans sa bibliothèque.
