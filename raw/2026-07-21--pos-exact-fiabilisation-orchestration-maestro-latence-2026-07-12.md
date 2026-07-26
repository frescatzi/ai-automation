---
type: raw
title: "POS-EXACT_Fiabilisation-orchestration-Maestro-latence_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_fiabilisation-orchestration-maestro-latence_2026-07-12.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Fiabilisation de l'orchestration Maestro (coût / latence)
**Date :** 2026-07-12 · **Projet :** LUMINA OS / AFTER SUN PEOPLE · **Scope :** AFTRSN-Maestro + sous-agents analytiques
**Objet :** Réduire drastiquement le temps/coût de Maestro (question stratégique : ~8 min 44 / ~242k tokens → ~20–30 s) tout en conservant la vérification qualité par les spécialistes.

## 0. Résultat (mesuré)
| Configuration | Décision vérifiée | Exec de référence |
|---|---|---|
| Départ (tous les sous-agents, Maestro Sonnet) | **~8 min 44 s / ~242k tokens** | #3907 |
| Conseil courant, réponse directe (0 sous-agent) | **~30 s / ~19,5k tokens** | #4046 |
| Décision, filtrée à 1 spécialiste (Maestro Sonnet) | ~53–104 s (bruité) | #4145 / #4189 / #4679 |
| **Décision, Maestro Haiku + spécialiste gpt-4o-mini** | **~20 s** | #4784 |

## 1. État FINAL live (source de vérité = connecteur n8n)
- **AFTRSN-Maestro** `OlrQO21u178SjgBK` — LLM = `@n8n/n8n-nodes-langchain.lmChatAnthropic`, modèle **`claude-haiku-4-5-20251001`** (Haiku 4.5). versionId `b18a3cd2…`. Prompt = 3 blocs de gouvernance (ci-dessous).
- **AFTRSN-Comptable-Finance** `F1wsXbsYZdEkwQ3n` — LLM = `@n8n/n8n-nodes-langchain.lmChatOpenAi`, modèle **`gpt-4o-mini`**. versionId `382759ac…`.
- **AFTRSN-Qualite-Compliance** `zbW4p5qsBYhkfBdi` — LLM = `@n8n/n8n-nodes-langchain.lmChatOpenAi`, modèle **`gpt-4o-mini`**. versionId `48a230bc…`.
- **Inchangés** : Marketing `wyUEojwZjSKFRaRL` + Channel-Content = Claude Sonnet (rédactionnel) ; Culture-Steward `vK6B2WuDfbse41Jv` + Experience-Designer `bBu5ycRwHoK7ACdV` = Gemini Flash ; Secretary `FtPFOKvi2t18DFb1` = GPT-mini.

## 2. Les 3 blocs insérés dans le system message de Maestro (dans l'ordre)

### 2.1 Bloc `## ÉCONOMIE` (anti-latence) — inséré après la ligne MEMORY
```
## ÉCONOMIE (anti-latence — obligatoire)
- Knowledge : 1 recherche MAX pour ton brouillon. Si le résultat n'est pas pertinent, N'INSISTE PAS (pas de 2e requête reformulée) — appuie-toi sur ton expertise.
- Pendant la RÉCONCILIATION : AUCUNE nouvelle recherche Knowledge.
- Pas de tours à vide : brouillon → (si enjeu) spécialiste(s) filtré(s) une fois → réconcilie une fois → livre.
```

### 2.2 Bloc `## SÉLECTION & VÉRIFICATION` — remplace la logique d'orchestration, avant ROUTING
```
## SÉLECTION & VÉRIFICATION (règle dure — VITESSE D'ABORD, cible 30–60 s)
Tu es toi-même l'expert (curation, événementiel, marque).
- CONSEIL COURANT / opinion sans enjeu → RÉPONDS DIRECTEMENT (ta propre expertise + AU PLUS UNE recherche Knowledge). 0 spécialiste. Cas NORMAL.
- VRAIE DÉCISION / ENJEU (argent, marque, action imminente, livrable engageant) → vérification :
   1. Rédige d'abord ton BROUILLON.
   2. Identifie les DOMAINES réellement touchés et n'envoie le brouillon QU'aux spécialistes de ces domaines (FILTRE strict — jamais un spécialiste « pour l'appeler »). culture→Culture-Steward · expérience/venue/flow→Experience-Designer · budget/prix/marge→Comptable-Finance · campagne/canal/angle→Marketing · conformité/risque→Qualité-Compliance. Souvent 1–2, 3 MAX, JAMAIS tous.
      Consigne à chaque spécialiste : « Restreins-toi à ta compétence. Grille courte : chaque affirmation → vérifiée/réfutée/non-vérifiable + correctif + source. »
   3. RÉCONCILIE : tout vérifié → livre (reformule SEULEMENT si un correctif l'exige) ; réfuté → 1 passe ciblée puis re-livre. CAP DUR 1–2 tours, sinon escalade.
- Doute sur l'enjeu → réponds directement et PROPOSE d'approfondir, ne lance pas de vérif d'office.
- La boucle Hermès ci-dessous ne concerne que l'EXÉCUTION factuelle (web/ops).
```

## 3. Marche à suivre EXACTE réalisée
1. **Lire l'état live** de Maestro via le connecteur (`get_workflow_details OlrQO21u178SjgBK`) + Bible §Maestro. Diagnostic : le prompt était entièrement bâti autour de la boucle Hermès, sans chemin « réponse directe » ni plafond de sous-agents → une question stratégique déclenchait TOUS les sous-agents.
2. **Éditer le prompt Maestro** dans le navigateur piloté (clics SPA only, login partagé) : ouvrir le node AFTRSN-Maestro, modifier le `<textarea>` du System Message via le DOM (setter natif `HTMLTextAreaElement.prototype.value` + `dispatchEvent(new Event('input'))`), fermer, **Publish** (bouton passe au jaune = modif captée), re-vérifier via le connecteur.
   - v2 : réponse directe par défaut (0 sous-agent) → exec #4046 = ~30 s, 0 sous-agent.
   - Étape 1 : bloc SÉLECTION & VÉRIFICATION (filtre par domaine + réconciliation) → exec #4145 = filtrage OK (seul Comptable-Finance appelé sur une décision prix), ~104 s.
   - Étape 2a : bloc ÉCONOMIE → exec #4189 = Knowledge 4→1, tours 6→4, ~99 s (gain faible : Maestro n'était pas le seul goulot).
3. **Basculer le modèle des spécialistes analytiques** (Comptable-Finance, Qualité-Compliance) : remplacer le node `lmChatOpenRouter` (Sonnet) par un node **OpenAI natif** (`lmChatOpenAi`) → sélectionner **`gpt-4o-mini`**. Méthode navigateur : sélectionner le node modèle → corbeille → cliquer le connecteur « Chat Model » de l'AI Agent → « OpenAI Chat Model » → credential OpenAI → modèle. Publish + vérif connecteur.
4. **Basculer Maestro sur Claude Haiku 4.5** (node `lmChatAnthropic`, `claude-haiku-4-5-20251001`). C'est le levier décisif : Maestro passe de 4 tours Sonnet (~61 s cumulés) à 2 tours Haiku (~10 s).
5. **Test final** #4784 (Maestro Haiku + Comptable gpt-4o-mini) : **~20 s**, 2 tours Maestro, 0 recherche Knowledge, 1 spécialiste filtré, réponse structurée sans hallucination.

## 4. Vérification
- Chaque publication re-vérifiée via `get_workflow_details` (versionId + updatedAt + valeur du champ). Les grosses sorties d'exécution (>50k tokens) parsées via sous-agent (jq/python) pour extraire : clés `runData` (= nodes exécutés → quels spécialistes appelés), nb de runs OpenAI/Knowledge, durée totale (max(startTime+executionTime) − min(startTime)), texte final.
- Comparaison de modèles pour le spécialiste (1 run chacun, à titre indicatif) : gpt-mini(OpenRouter) ~53 s · o4-mini(natif) ~78 s · gpt-5-mini(natif) ~100 s → **gpt-4o-mini retenu** (mini non-raisonneur = le plus rapide, qualité de challenge équivalente).

## 5. Rollback
Chaque workflow a son historique de versions n8n → **republier la version précédente** annule tout (Maestro : revenir à un versionId antérieur ; spécialistes : idem). Aucune donnée détruite, aucun câblage d'outil modifié (seuls le prompt Maestro et les nodes modèle ont changé).

## 6. Difficultés rencontrées
- **UI n8n qui gèle** en rendant une exécution lourde (chat/logs) → screenshots et JS injectés en timeout.
- **Session fragile** : seul l'onglet piloté d'origine garde la session ; tout onglet neuf / rechargement d'URL renvoie « Unauthorized » ou vers `/signin`.
- **Perte de connexion websocket** (« Connection lost ») en cours d'édition, bouton Publish masqué.
- **Gain de latence trompeur** : dégraisser Maestro (moins de Knowledge/tours) n'a rendu que ~5 s ; le vrai coût était ailleurs (tours Sonnet de Maestro + le spécialiste).
- **Latence LLM très bruitée** : le même chemin a donné 53 s puis 100 s selon les runs.

## 7. Solutions implémentées
- Édition via **DOM** (setter + event input) pour le textarea récalcitrant ; navigation **SPA uniquement** (barre de commande ⌘K), jamais de rechargement d'URL.
- **Connecteur = source de vérité** pour valider chaque publication (indépendant du navigateur gelé).
- **Sous-agent** pour parser les grosses sorties d'exécution hors du contexte principal.
- Après « Connection lost » : rafraîchir + se reconnecter dans l'onglet, re-appliquer l'édition (le texte exact était connu), republier.
- Attaquer le **vrai levier** : modèle + nombre de tours de Maestro (Haiku + bloc ÉCONOMIE), et modèle rapide non-raisonneur pour les spécialistes.

## 8. Lessons learned (pièges figés)
- **Le goulot d'une orchestration, c'est l'ORCHESTRATEUR** (son modèle + son nombre de tours), pas les sous-agents. Optimiser Maestro d'abord.
- **Modèles « raisonneurs » (o4-mini, gpt-5-mini) = plus lents** pour une tâche de challenge courte ; préférer un mini **non-raisonneur** (`gpt-4o-mini`).
- **Répartition des modèles** : orchestrateur/voix finale = modèle rédactionnel rapide (Haiku) ; vérificateurs analytiques = mini rapide (gpt-4o-mini) ; rédactionnels lourds = Sonnet.
- **Comparer des modèles sur 1 run n'est pas fiable** (latence bruitée) → A/B en 5–10 runs si on veut trancher au chiffre près.
- **n8n** : DOM pour les champs récalcitrants · connecteur pour la vérité · SPA-only (jamais recharger l'URL = perte de session) · parser les exécutions lourdes en sous-agent.
- Compromis Haiku : ~20 s mais un cran sous Sonnet en finition éditoriale (léger mélange vous/tu, a sauté le Save Episode une fois) → acceptable pour le quotidien, repasser Maestro sur Sonnet ponctuellement pour une réponse à fort enjeu de voix.
