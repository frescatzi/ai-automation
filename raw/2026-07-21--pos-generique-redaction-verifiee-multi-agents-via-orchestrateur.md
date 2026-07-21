---
type: raw
title: "POS-GENERIQUE_Redaction-verifiee-multi-agents-via-orchestrateur"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_redaction-verifiee-multi-agents-via-orchestrateur.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Rédaction vérifiée par un orchestrateur multi-agents (patron réutilisable)

**Objet :** faire produire par un bot un **livrable rédactionnel engageant** (email, message, post…) qui **respecte la voix et les règles d'une marque**, en réutilisant un **orchestrateur multi-agents existant** (rédacteur + gardiens qualité) plutôt qu'un simple node LLM. Réutilisable pour n'importe quelle marque / n'importe quel canal.

## 1. But
Depuis un sous-workflow « canal » (Gmail, Telegram, etc.), au lieu d'appeler un LLM figé, **déléguer la rédaction à l'orchestrateur de la marque** qui : fait rédiger par l'agent rédacteur, fait **relire en parallèle** par les gardiens (voix + conformité), réconcilie, journalise (mémoire/apprentissage), et renvoie **uniquement** le livrable final structuré. Le canal reste mince et **ne fait qu'une action réversible** (créer un brouillon), jamais l'envoi.

## 2. Préconditions
- Un orchestrateur appelable **en sous-workflow** (trigger « Executed by Another Workflow ») avec une boucle rédaction→vérification→réconciliation déjà câblée.
- Des agents spécialistes distincts par **domaine** (garde-voix, conformité/PII…) + outils de mémoire (journal + leçons).
- Accès édition au sous-workflow canal (via store de l'éditeur si l'API n'édite pas).
- Historique de versions (rollback).

## 3. Étapes (à partir de 1)
1. **Tracer l'état live** de l'orchestrateur et des agents (modèle, prompt, sous-agents réellement câblés, statut actif). Ne rien supposer depuis la doc.
2. **Node « Prepare brief »** (Code) : construit une instruction qui (a) marque la tâche comme *livrable engageant* pour déclencher la vérification, (b) impose une **règle de complétion** (« appelle réellement les outils, attends les verdicts, réconcilie, PUIS réponds »), (c) **interdit** tout message d'avancement, (d) paramètre voix/ton par segment, (e) demande la journalisation (épisode + leçon si correctifs réutilisables), (f) fixe un **format de sortie strict** (objet JSON unique).
3. **Node « Call orchestrateur »** (Execute Sub-workflow, **Wait for completion ON**) : passe `{query, chat_id}`.
4. **Node « Parse »** (Code) : extraction **robuste** du livrable — chercher le **dernier** objet JSON valide (blocs de code + balayage à accolades équilibrées), **filtrer les placeholders**, émettre un flag `ok`.
5. **Garde-fou** (IF sur `ok`) : succès → action réversible (créer un brouillon) ; échec → **message d'échec propre, aucune action**.
6. **Préserver l'invariant** : le canal ne fait que du brouillon ; l'envoi/action critique reste **gaté** par une porte de validation humaine.
7. **Publier** puis **recharger + re-vérifier** l'état persisté.

## 4. Vérification
- Test isolé de l'orchestrateur (via API/chat) : confirmer que **tous** les agents attendus tournent (rédacteur + gardiens) et que la sortie contient le livrable.
- Test bout-en-bout depuis le canal réel : livrable propre reçu **ou** message d'échec propre (jamais d'action sur sortie non conforme).
- Non-régression des autres branches du canal.

## 5. Rollback
Republier la version précédente du sous-workflow canal (historique de versions). L'orchestrateur et les agents ne sont pas modifiés.

## 6. Difficultés rencontrées (typiques)
- Orchestrateur sur **modèle rapide** = risque de « narrate-and-stop » (décrit au lieu d'exécuter), **non déterministe**.
- LLM qui **emballe** le JSON (fences markdown) et répète un **placeholder** → parsing fragile.
- Édition de l'éditeur no-code : mutation programmatique qui **n'est pas persistée** sans vrai événement d'UI.
- Sur échec, le canal peut créer un **artefact poubelle** (brouillon vide/incohérent).

## 7. Solutions implémentées (typiques)
- **Instruction de complétion** + **interdiction explicite** des messages d'avancement.
- **Parseur tolérant** : dernier objet JSON valide, filtrage placeholders, flag de succès.
- **Garde-fou aval** : aucune action si la sortie n'est pas un vrai livrable.
- **Persistance** : muter puis **salir par un vrai événement** avant publication ; se fier à l'indicateur d'UI réel, pas au flag interne ; **recharger + re-vérifier**.
- **Journalisation explicite** dans le brief (épisode + leçon) pour nourrir l'apprentissage de l'exécuteur central.

## 8. Lessons learned (transférables)
- **Réutiliser l'orchestrateur > réimplémenter** la vérification : moins de code, cohérence, et l'exécuteur apprenant **capitalise** (leçons) à chaque tâche.
- **Ceinture + bretelles** avec un orchestrateur LLM rapide : instruction de complétion **ET** garde-fou déterministe en aval.
- **Ne jamais faire confiance au format** d'un LLM : parser défensivement, filtrer les placeholders.
- **Séparer rôle et exécuteur** : le rédacteur/gardiens sont des rôles ; à terme leur exécution peut migrer sur un exécuteur central apprenant — mais **ne le mettre sur le chemin critique synchrone qu'une fois sa latence/fiabilité prouvée**.
- **Tracer l'état live d'abord** : la réalité de la plateforme est souvent en avance sur la doc.
