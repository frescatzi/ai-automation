---
type: raw
title: "POS-GENERIQUE_Health-check-workflow-live-n8n-API-interne"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_health-check-workflow-live-n8n-api-interne.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GENERIQUE — Health-check d'un workflow n8n live (via API interne) + réglage sûr
**Scope :** réutilisable pour tout workflow n8n hors portée du connecteur, toute marque.

## But
Vérifier qu'un workflow live est complet et sain (structure, câblage, exécutions, test bout-en-bout) et appliquer un réglage (ex. Error Workflow) sans risque, en lecture seule d'abord.

## Préconditions
- Navigateur connecté à l'instance n8n (session valide) ; `browser-id = localStorage['n8n-browserId']`.
- Requêtes `fetch` en `credentials:'include'` + header `browser-id`.

## Étapes (numérotées à partir de 1)
1. **Confirmer la session** : `GET /rest/login` → 200 + email attendu.
2. **Lire la structure** : `GET /rest/workflows/:id` → `active`, `nodes`, `connections`, `settings` (dont `errorWorkflow`). Extraire les params des nodes clés (URL/body HTTP, modèle LLM, conditions IF, cibles executeWorkflow).
3. **Vérifier le câblage** : suivre `connections` de bout en bout (le node de dispatch pointe bien vers la bonne cible / le bon sous-workflow).
4. **Contrôler la santé** : `GET /rest/executions?filter={"workflowId":"…"}&limit=N` → statut succès/erreur + latence des runs récents.
5. **Test cœur sans effet de bord** : appeler le webhook interne du service (`POST /webhook/…`) avec un payload de test → vérifier le code HTTP + la forme de la réponse.
6. **Test bout-en-bout** (si effet de bord accepté) : déclencher le vrai chemin (ex. message réel) → relire l'exécution ; décoder le **flatted** de `/rest/executions/:id?includeData=true` pour extraire la sortie exacte.
7. **Réglage sûr** : `PATCH /rest/workflows/:id` avec le **seul champ concerné** (ex. `{settings:{…, errorWorkflow:"…"}}`) → applique en place. Éviter de renvoyer les `nodes` (filtre de sécurité sur `=`,`?`,`&`).
8. **Vérifier** par relecture (`GET`) : réglage persistant, `active` conservé.

## Vérification
- État live conforme (structure + câblage + exécutions vertes) ; réglage appliqué et persistant.

## Rollback
- Re-PATCH le champ à sa valeur précédente.

## Difficultés rencontrées (typiques)
- Workflow **hors portée du connecteur** → API interne obligatoire.
- Données d'exécution en **flatted** (illisibles brutes).
- PATCH complet risqué (filtre de sécurité sur le contenu des expressions).

## Solutions implémentées (typiques)
- API interne `browser-id` + cookies ; **PATCH ciblé** (champ seul) ; **décodeur flatted** ; lecture seule d'abord, mutation ensuite.

## Lessons learned (pièges figés)
- **PATCH le champ minimal** (souvent `settings`) pour éviter de toucher aux nodes et au WAF ; s'applique en place.
- **Décoder le flatted** avant de lire une exécution (nombres inline ; chaînes = index de slot).
- **Un webhook interne** permet un test cœur **sans effet de bord** avant tout test bout-en-bout.
- **Grounding strict** → une réponse « pas trouvé » malgré des extraits = souci de récupération, pas de plomberie.

*POS-GENERIQUE — procédure abstraite, réutilisable.*
