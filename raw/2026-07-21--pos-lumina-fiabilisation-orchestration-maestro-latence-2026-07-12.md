---
type: raw
title: "POS-LUMINA_Fiabilisation-orchestration-Maestro-latence_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_fiabilisation-orchestration-maestro-latence_2026-07-12.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-LUMINA — Orchestration Maestro : sélection filtrée, vérification, latence maîtrisée
**Date :** 2026-07-12 · **Criticité :** 🔴 (Maestro = zone rouge) · **Statut :** en service

## But
Garantir que l'orchestrateur AFTRSN-Maestro : (1) répond directement aux questions courantes ; (2) sur une vraie décision à enjeu, ne consulte que les spécialistes des domaines touchés puis réconcilie ; (3) tient une latence cible (~20–30 s en usage courant).

## Préconditions
1. Session n8n active dans l'onglet piloté (login partagé) — édition en clics SPA uniquement.
2. Connecteur n8n disponible en lecture (source de vérité).
3. Credentials en place : OpenAI (pour les sous-agents `gpt-4o-mini`), Anthropic (pour Maestro Haiku), OpenRouter (agents restés dessus).

## Procédure (étapes numérotées à partir de 1)
1. Ouvrir AFTRSN-Maestro (`OlrQO21u178SjgBK`), node « AFTRSN-Maestro », onglet System Message.
2. Vérifier la présence des 3 blocs de gouvernance : `## ÉCONOMIE`, `## SÉLECTION & VÉRIFICATION`, section `SPECIALISTS:`. Si absents, les insérer (textes de référence dans le POS-EXACT du 2026-07-12).
3. Vérifier le node modèle de Maestro = `lmChatAnthropic` / `claude-haiku-4-5-20251001`.
4. Pour chaque sous-agent **analytique/vérificateur** (Comptable-Finance, Qualité-Compliance) : node modèle = `lmChatOpenAi` / `gpt-4o-mini`.
5. Laisser inchangés : Marketing + Channel-Content (Sonnet, rédactionnel) ; Culture-Steward + Experience-Designer (Gemini Flash) ; Secretary (GPT-mini).
6. Publier chaque workflow modifié (bouton Publish → jaune = capté).
7. Re-vérifier chaque publication via `get_workflow_details` (versionId + valeurs).

## Vérification
- Test « conseil courant » (ex. « penses-tu qu'on devrait faire un after-party cet automne ? ») → réponse directe, **0 sous-agent**, ~30 s.
- Test « décision à enjeu » (ex. « je fixe le billet à 50€, marge correcte ? à vérifier ») → **1 seul spécialiste** filtré (Comptable-Finance), réconciliation, ~20 s, réponse structurée sans chiffre inventé.
- Confirmer via les `runData` de l'exécution : seuls les nodes attendus apparaissent (pas de fan-out).

## Rollback
Republier la version n8n précédente du/des workflow(s) concerné(s). Aucune donnée ni câblage d'outil impacté.

## Difficultés rencontrées
- UI n8n qui gèle sur les exécutions lourdes ; session limitée à l'onglet d'origine ; « Connection lost » en cours d'édition.
- Faux signal : dégraisser Maestro n'a rendu que ~5 s ; le levier réel était son modèle (Sonnet→Haiku) et le modèle des spécialistes.
- Latence LLM bruitée (53 s → 100 s pour le même chemin selon les runs).

## Solutions implémentées
- Édition DOM (setter + event), navigation SPA-only, connecteur comme source de vérité, parsing des grosses exécutions en sous-agent.
- Après coupure : refresh + relogin dans l'onglet, ré-application, republication.
- Optimisation ciblée sur le vrai goulot : Maestro→Haiku + bloc ÉCONOMIE ; spécialistes→gpt-4o-mini.

## Lessons learned
- **L'orchestrateur est le goulot** : son modèle + son nombre de tours priment sur les sous-agents.
- **Mini non-raisonneur (gpt-4o-mini) > raisonneurs (o4-mini, gpt-5-mini)** pour un challenge court et rapide.
- **Bon modèle au bon rôle** ; **A/B multi-runs** pour tout choix de modèle définitif ; **numérotation à partir de 1**.
- **Invariant** : Maestro reste zone rouge — vérifier la valeur live avant toute modif, éditer de façon ciblée, republier, re-vérifier.
