---
type: raw
title: "POS-EXACT_Health-check-M3-Memory-Search_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_health-check-m3-memory-search_2026-07-12.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-EXACT — Health-check M3 Memory-Search + branchement Error Workflow
**Date :** 2026-07-12 · **Scope :** LUMINA OS / Lyra · **Exécutant :** Claude (Cowork) · **Accès :** API interne n8n (`/rest/*`, session itiskarter, header `browser-id`)

## But
Vérifier en live que le module **M3 (Memory-Search)** de Lyra est complet et sain, corriger le seul écart trouvé (Error Workflow), sans rien construire de neuf.

## Préconditions
- Navigateur connecté à **n8n.aftersunpeople.com** en tant qu'itiskarter@gmail.com (`/rest/login` = 200).
- `browser-id` = `localStorage['n8n-browserId']` (ici `caa44edc…`).
- Workflows hors portée du connecteur n8n (projet Personal) → lecture/écriture par l'API interne.

## Étapes réalisées (numérotées à partir de 1)
1. **Session vérifiée** : `GET /rest/login` → 200, email itiskarter@gmail.com.
2. **Memory-Search lu** : `GET /rest/workflows/AzEakDoVK7T9EXG2` → **actif**, nodes conformes, contrat `{question, brand, limit:5}` → `/webhook/memory-search`, LLM `gpt-5-mini` grounded. **`settings.errorWorkflow = null`** (écart).
3. **Gateway lu** : `GET /rest/workflows/uGF3pSv6aTK79cqi` (v `959ef7c8`, actif) → dispatch câblé `Question for Memory-Search ?` (intent `memory_search`/`general_question`) → `Prepare the search` → `Perform memory search` (executeWorkflow → `AzEakDoVK7T9EXG2`) → `Reply to user`. `errorWorkflow = xzaH0uWy0idKVphF`. **Sentinel actif** (`GET` → `active:true`).
4. **Exécutions contrôlées** : `GET /rest/executions?filter={workflowId}` → appels intégrés récents de Memory-Search (#5278, #5295) **succès** (~9–11 s) ; Gateway récents **succès**.
5. **Test cœur (sans effet de bord)** : `POST /webhook/memory-search {question, brand:"lumina", limit:5}` → **200**, 5 extraits pertinents (SYSTEM-CANON, synthèse), similarité ~0,47.
6. **Test bout-en-bout live** : Karter envoie « /memory lumina qu'est-ce que Hermes Exec ? » à @luminaLyraBot → Gateway **#5393** succès (ACL→/help?→Lire+purger état→Classify=`memory_search`→Perform→Reply) → Memory-Search **#5395** succès → Telegram **`ok:true`, `message_id 102`**. Réponse grounded prudente (anti-hallucination OK). Lecture des réponses via décodage **flatted** de `/rest/executions/:id?includeData=true`.
7. **Fix appliqué** : `PATCH /rest/workflows/AzEakDoVK7T9EXG2` avec **`settings` seul** = `{…, errorWorkflow:"xzaH0uWy0idKVphF"}` → 200, `active:true` conservé, versionId inchangé. **Relecture** : `errorWorkflow` = `xzaH0uWy0idKVphF` ✓.

## Vérification
- `AzEakDoVK7T9EXG2` : `active:true`, `settings.errorWorkflow = "xzaH0uWy0idKVphF"`.
- Bout-en-bout : Gateway #5393 + Memory-Search #5395 succès, Telegram `message_id 102`.

## Rollback
- Remettre `settings.errorWorkflow = null` via le même PATCH `settings`-seul si nécessaire.

## Difficultés rencontrées
- Workflows M3/Gateway **invisibles au connecteur n8n** (projet Personal) → « Workflow not found ».
- `settings.errorWorkflow` **absent** sur Memory-Search alors que la Bible v1.9 l'affirmait présent.
- Données d'exécution au format **`flatted`** (références par index), non lisibles directement.
- PATCH d'un workflow live = risque de casser les nodes / de buter sur le filtre de sécurité (WAF sur `=`,`?`,`&`).

## Solutions implémentées
- Lecture/écriture via **API interne** (`browser-id` + cookies).
- **PATCH du seul `settings`** (pas les nodes) → applique en place, reste actif, évite le WAF.
- Décodeur **flatted** maison pour extraire la réponse exacte des exécutions.
- Correction de l'écart doc dans la Bible (v2.3).

## Lessons learned (pièges figés)
- **PATCH `settings`-seul** = changer un réglage (Error Workflow…) sans toucher aux nodes ; s'applique en place (pas d'`activate`).
- **`/rest/executions/:id?includeData=true` renvoie du flatted** → décoder (nombres inline, chaînes = index de slot).
- **Grounding strict = prudence** : une réponse « pas trouvé » malgré des extraits = problème de **récupération/brand**, pas de plomberie M3.
- **n8n fait foi** : la Bible v1.9 se trompait sur l'Error Workflow ; toujours vérifier live.

*POS-EXACT — 2026-07-12. Double-sauvegardé : projet Claude + Drive `LUMINA AI DOCS`.*
