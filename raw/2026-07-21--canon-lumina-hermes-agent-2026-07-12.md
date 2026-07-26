---
type: raw
title: "CANON_LUMINA-Hermes-Agent_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/canon_lumina-hermes-agent_2026-07-12.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# LUMINA-Hermes-Agent (ex « Hermes-Exec ») — fiche canon

**Alias / anciens noms :** LUMINA-Hermes-Agent = **ex-LUMINA-Hermes-Exec** = « Hermes Exec » = « Hermes Ops ». Workflow n8n `Dwv4rcMqNAyQzlrF` (renommé Exec → Agent le 2026-07-11). Toute mention de « Hermes-Exec » désigne ce même composant.

## Définition (en une phrase)
Hermes-Agent est le **bras d'exécution apprenant** de LUMINA OS : c'est le composant qui **fait réellement les tâches** (recherche web, opérations) en s'appuyant sur un moteur Hermes auto-hébergé, et qui **apprend** en réutilisant les procédures déjà éprouvées.

## À quoi il sert
- **Exécuter** des tâches d'opérations et de recherche que les agents décident mais ne font pas eux-mêmes (les agents raisonnent / rédigent ; Hermes agit).
- **Apprendre** : avant chaque tâche, il recherche par similarité les *skills* pertinents (procédures apprises, collection `skills`) dans les banques `lumina` + `<marque>_memory` (seuil de distance < 0.35) et les injecte dans son contexte ; il **journalise** l'usage des skills (boucle d'amélioration continue, consolidée la nuit).

## Comment il fonctionne
- Reçoit une tâche `{ message }` (via `On Workflow Call` pour un autre workflow, ou `On Chat Message` en test).
- Prépare l'entrée → recherche skills (Postgres) → **exécute via le moteur Hermes** self-hosted (`hermes.aftersunpeople.com`, Coolify) en **cookie-auth** (credential Custom Auth, jamais de mot de passe en clair) → journalise (`Log Skills`) → renvoie le résultat.
- Le moteur Hermes répond actuellement via un gateway modèle (famille Claude Opus) ; cible future = modèle local.

## Qui l'appelle
- Sous-agents **Secretary** et **Marketing** via l'outil `Call 'Hermes (Ops)'`.
- Le bot **Lyra** : chemin **M4 Web/Hermes** (`LUMINA-TELEGRAM-WEB-HERMES` `RRAZbO4zEb1sFNvb`) pour les intentions `web_search` / `hermes_ops`.

## Repères techniques
- Workflow : `Dwv4rcMqNAyQzlrF` — nom live **LUMINA-Hermes-Agent** — actif — Error Workflow = Sentinel `xzaH0uWy0idKVphF`.
- Contrat d'entrée : `{ message }`. Auth moteur : cookie-auth (Custom Auth). Sécurité : mot de passe uniquement dans la credential, jamais dans un node/prompt/log.
- Distinct du **Router** (`LUMINA-AI-Router`, choix du LLM par `task_type`) et du **Maestro** (orchestrateur) : Hermes = exécution, pas orchestration ni choix de modèle.

*Fiche canon — 2026-07-12. But : donner une définition nette et à jour de Hermes-Agent (ex-Hermes-Exec) pour que la recherche mémoire réponde clairement, quel que soit le terme employé.*
