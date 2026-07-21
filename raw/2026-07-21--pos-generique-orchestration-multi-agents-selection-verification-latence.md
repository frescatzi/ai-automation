---
type: raw
title: "POS-GENERIQUE_Orchestration-multi-agents-selection-verification-latence"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_orchestration-multi-agents-selection-verification-latence.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GENERIQUE — Fiabiliser un orchestrateur multi-agents (sélection, vérification, latence)
**Objet :** Rendre un agent orchestrateur (hub) rapide et économe SANS perdre la vérification qualité par des sous-agents spécialistes. Réutilisable pour n'importe quelle marque / n'importe quel roster d'agents.

## 1. But
Un orchestrateur qui, par défaut, appelait trop de sous-agents (lent, cher) doit :
- répondre **directement** aux questions courantes (0 sous-agent) ;
- sur une **vraie décision à enjeu**, ne consulter QUE les spécialistes des **domaines réellement touchés** (filtre strict + plafond), puis **réconcilier** et livrer ;
- tenir une **cible de latence** explicite.

## 2. Préconditions
- Accès en lecture à la config live des workflows (API/connecteur) = source de vérité.
- Accès en édition (navigateur piloté si l'API n'édite pas ce scope).
- Un historique de versions par workflow (rollback).
- Connaître les modèles disponibles et leur profil (rapide/non-raisonneur vs raisonneur vs rédactionnel).

## 3. Étapes (numérotées à partir de 1)
1. **Diagnostiquer sur le prompt live**, pas sur la doc. Chercher : y a-t-il un chemin « réponse directe » ? un plafond de sous-agents ? une règle de sélection par domaine ? Souvent l'absence de ces trois crée le fan-out.
2. **Ajouter une règle de SÉLECTION & VÉRIFICATION** au prompt de l'orchestrateur :
   - question courante → réponse directe, 0 spécialiste ;
   - décision à enjeu → brouillon → identifier les domaines touchés → n'appeler QUE les spécialistes correspondants (table domaine→spécialiste) → **plafond dur** (ex. 3 max, jamais tous) → réconcilier (1 passe de correction max) → livrer.
3. **Ajouter une règle d'ÉCONOMIE** : 1 seule recherche mémoire (RAG) pour le brouillon ; aucune re-recherche pendant la réconciliation ; pas de tours à vide.
4. **Mesurer** (1 exécution représentative) : décomposer le temps par node (orchestrateur vs sous-agents vs RAG). **Identifier le vrai goulot.**
5. **Optimiser le goulot dans l'ordre de l'impact :**
   a. **Modèle + nombre de tours de l'ORCHESTRATEUR** (levier dominant) : un modèle rédactionnel **rapide** (type « Haiku »/léger de la même famille) coupe souvent la moitié du temps ;
   b. **Modèle des sous-agents analytiques/vérificateurs** : un **mini non-raisonneur** (type « gpt-4o-mini ») ; éviter les modèles « raisonneurs » (plus lents) pour un challenge court ;
   c. garder un **modèle rédactionnel fort** (type « Sonnet ») uniquement là où la qualité de rédaction prime.
6. **Re-mesurer** après chaque changement ; comparer sur **plusieurs runs** (la latence LLM est bruitée — 1 run ne prouve rien).
7. **Verrouiller** la config gagnante et **documenter** (exact + générique + SOP).

## 4. Vérification
- Chaque édition re-lue via l'API/connecteur (version + valeur du champ).
- Test bout-en-bout : mesurer la durée ET vérifier quels sous-agents ont réellement tourné (le filtrage marche-t-il ?) ET la qualité de la réponse finale.

## 5. Rollback
Republier la version précédente du workflow (historique de versions). Ne toucher que le prompt et les nodes modèle → aucun câblage d'outil impacté.

## 6. Difficultés rencontrées (typiques)
- Le gain attendu d'un dégraissage de l'orchestrateur peut être **faible** si le vrai coût est ailleurs (sous-agent lourd, modèle raisonneur).
- Latence LLM **bruitée** : comparer des modèles sur un seul run induit en erreur.
- Éditeur récalcitrant / session fragile côté outil d'édition.

## 7. Solutions implémentées (typiques)
- Toujours **décomposer le temps par node** avant d'optimiser → attaquer le vrai goulot.
- Édition robuste via le DOM ; source de vérité = l'API, pas l'UI.
- A/B multi-runs pour trancher un choix de modèle.

## 8. Lessons learned (transférables)
- **Optimise l'orchestrateur avant les sous-agents** : son modèle et son nombre de tours dominent la latence.
- **Filtrer par domaine** réduit souvent à 1–2 sous-agents → le parallélisme devient secondaire.
- **Bon modèle au bon rôle** : rédactionnel rapide pour la voix finale ; mini non-raisonneur pour la vérification ; modèle fort seulement pour la rédaction lourde.
- **Vitesse ≠ un chiffre fixe** : c'est une fourchette ; ne jamais conclure sur 1 run.
- **Ne pas sur-optimiser** : une décision rare à fort enjeu peut légitimement coûter un peu de temps si elle est réellement vérifiée et sans hallucination.
