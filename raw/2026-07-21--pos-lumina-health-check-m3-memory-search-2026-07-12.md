---
type: raw
title: "POS-LUMINA_Health-check-M3-Memory-Search_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_health-check-m3-memory-search_2026-07-12.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-LUMINA — Health-check M3 Memory-Search (Lyra)
**Scope :** LUMINA OS / Lyra · **Date :** 2026-07-12

## But
S'assurer que M3 (Memory-Search) répond de bout en bout et alerte en cas d'erreur.

## Préconditions
1. Session n8n itiskarter@gmail.com dans le navigateur (`/rest/login` = 200) + `browser-id`.
2. IDs : Memory-Search `AzEakDoVK7T9EXG2` · Gateway `uGF3pSv6aTK79cqi` · Sentinel `xzaH0uWy0idKVphF` · webhook `/webhook/memory-search` · owner chat_id `776345147`.

## Étapes (numérotées à partir de 1)
1. **Memory-Search** : `GET /rest/workflows/AzEakDoVK7T9EXG2` → actif ; contrat `{question, brand, limit:5}` ; LLM `gpt-5-mini` grounded ; **vérifier `settings.errorWorkflow`** (doit = `xzaH0uWy0idKVphF`).
2. **Gateway** : dispatch `Question for Memory-Search ?` (intent `memory_search`/`general_question`) → `Perform memory search` (executeWorkflow → `AzEakDoVK7T9EXG2`) → `Reply to user` ; `errorWorkflow` = Sentinel ; Sentinel actif.
3. **Exécutions** : runs intégrés récents de Memory-Search + Gateway = succès.
4. **Test cœur** : `POST /webhook/memory-search {question, brand:"lumina", limit:5}` → 200 + extraits.
5. **Test bout-en-bout** : message réel à @luminaLyraBot → relire Gateway + Memory-Search (succès) + Telegram `ok:true` ; décoder le flatted pour la réponse exacte.
6. **Si `errorWorkflow` absent** : `PATCH /rest/workflows/AzEakDoVK7T9EXG2` `{settings:{…,errorWorkflow:"xzaH0uWy0idKVphF"}}` (settings seul) → relire.

## Vérification
- `AzEakDoVK7T9EXG2` actif + `errorWorkflow = xzaH0uWy0idKVphF` ; test live Telegram `ok:true` (ex. `message_id 102` le 12-07, exécutions Gateway #5393 / Memory-Search #5395).

## Rollback
- Re-PATCH `settings.errorWorkflow` à sa valeur antérieure.

## Difficultés rencontrées
- M3/Gateway hors connecteur ; `errorWorkflow` manquant sur Memory-Search (écart vs Bible v1.9) ; exécutions en flatted.

## Solutions implémentées
- API interne ; PATCH `settings`-seul (en place, reste actif) ; décodeur flatted ; correction Bible v2.3.

## Lessons learned (pièges figés)
- **PATCH settings-seul** pour un réglage sans toucher aux nodes (évite le WAF `=`,`?`,`&`).
- **Décoder le flatted** pour lire une exécution.
- **n8n fait foi** : vérifier live (la Bible v1.9 se trompait sur l'Error Workflow).
- **Grounding prudent** → réponse pauvre = tuning récupération (contenu/brand), pas plomberie.

*POS-LUMINA — 2026-07-12. Double-sauvegardé : projet Claude + Drive `LUMINA AI DOCS`.*
