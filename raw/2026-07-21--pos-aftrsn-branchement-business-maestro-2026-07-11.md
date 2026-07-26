---
type: raw
title: "POS-AFTRSN_Branchement-Business-Maestro_2026-07-11"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_branchement-business-maestro_2026-07-11.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-AFTRSN — Branchement Business → Maestro (Telegram)

**Date :** 2026-07-11 · **Statut :** construit & publié (⚠ bug clarification à corriger, voir §Suivi) · **Boussoles :** Spec Build Lumina Telegram + Lumina Bible 360.

## Objet
Faire aboutir une question business/décision reçue sur Telegram (bot Lyra) jusqu'à l'orchestrateur **AFTRSN-Maestro** et renvoyer sa réponse. Détail clic-par-clic → voir **POS-EXACT_Branchement-Business-Maestro_2026-07-11.md**.

## Architecture livrée
`Gateway → Classifieur d'intention (Router) → … → Is it a web/ops question? (FALSE) → Is it Maestro? → (true) Call 'AFTRSN-Maestro' → Reply (maestro) ; (false) Reply (module pending).`

## Invariants (à ne jamais casser)
1. **Router** : intent `maestro` présent dans l'enum (séparateur `|`). Règle business/décision → maestro.
2. **Maestro** : DEUX triggers (Chat + « When Executed by Another Workflow »/Accept all data) branchés sur l'entrée de l'agent ; prompt en « Define below » = `{{ $json.chatInput || $json.query || $json.message }}`. Sortie = champ **`output`**.
3. **Gateway** : `Reply (maestro)` = Telegram sendMessage, credential **`LUMINA-Lyra - Telegram`**, `chat_id = {{ $('Classify the intent').first().json.chat_id }}`, `text = {{ $json.output }}`.
4. Le node `Call 'AFTRSN-Maestro'` = Execute Sub-workflow, « Run once with all items », cible `OlrQO21u178SjgBK`.
5. Aucune double-alimentation de `Reply (module pending)` (l'ancien lien web/ops-FALSE a été supprimé).

## IDs
Router `KTLwQi7ZKHDTexMZ` · Maestro `OlrQO21u178SjgBK` · Gateway `uGF3pSv6aTK79cqi` · owner chat_id `776345147`.

## Criticité
**PROD.** Le Gateway est actif et sert le bot Lyra. Toute édition se fait **en clics SPA** dans l'onglet piloté connecté ; vérifier les connexions par le DOM avant Publish ; n8n conserve les versions (rollback possible).

## Vérification (avant de considérer OK)
1. Maestro exécuté isolément (chatInput business) → `output` non vide. **[OK 11-07, exec 3826]**
2. Table des 6 connexions vérifiée (DOM). **[OK]**
3. Test réel Telegram question business. **[⚠ révèle le bug clarification, voir Suivi]**

## Suivi / correctif requis (PRIORITÉ 1, prochaine session)
Le classifieur émet `intent=maestro` **ET** `needs_clarification=true` (marque « none ») → la porte « Need clarification? », traversée **avant** le routage, fait boucler la demande de marque. **Fix :** dans le Router, forcer `needs_clarification=false` quand `intent="maestro"` (Maestro gère la marque ; défaut `aftrsn`), publier, re-tester. Cause confirmée via exécution Gateway `3832`.

## Difficultés / Solutions / Lessons learned
- **Difficulté :** navigateur piloté perd la session n8n au rechargement d'URL. **Solution :** login dans l'onglet piloté + clics SPA only. **Lesson :** jamais de full navigate.
- **Difficulté :** enum réel séparé par `|` (≠ export). **Solution :** modif ciblée de la valeur live. **Lesson :** vérifier la valeur live.
- **Difficulté :** champ non committé (popup). **Solution :** setter natif + fermer/rouvrir pour vérifier. **Lesson :** toujours re-vérifier.
- **Difficulté :** câblage IF au pixel fragile. **Solution :** connexions vérifiées via DOM. **Lesson :** DOM = vérité.
- **Difficulté :** porte de garde (clarification) avant routage → boucle. **Solution (à venir) :** exempter maestro de la clarification. **Lesson :** un nouvel intent doit être exempté des portes de garde si l'agent gère lui-même le contexte.

## Livraison documentaire
Ce POS + POS-EXACT + POS-GENERIQUE, double-sauvegardés (projet Claude + Drive `LUMINA AI DOCS`). Addendum Bible 360 + coche Spec Build à faire.
