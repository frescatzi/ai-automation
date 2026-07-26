---
type: raw
title: "POS-GENERIQUE_recette-cloture-verification-finale-moteur-multi-workflow_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_recette-cloture-verification-finale-moteur-multi-workflow_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-GÉNÉRIQUE · Recette : vérification finale et clôture d'un moteur multi-workflow

Date : 20.07.2026 · Contexte d'origine : AFTRSN Phase 3, milestone M9. Recette réutilisable pour clôturer un ensemble de workflows d'automatisation (n8n ou équivalent) piloté par un backlog.

## Quand l'appliquer

À la fin d'un chantier composé de plusieurs workflows chaînés, avant de déclarer une phase « close » : on veut prouver que tout tourne, que le filet d'erreur est branché partout, et que la chaîne fonctionne de bout en bout, puis sceller la documentation.

## Patron en 4 temps

1. **Inventaire d'état par API (lecture seule).** Health check de l'instance. Lister les workflows et confirmer que chaque brique attendue est `active`. Sur un gros parc, ne pas lire tout : sauvegarder la sortie et filtrer par les IDs connus du chantier.
2. **Audit du filet d'erreur.** L'état `active` ne dit pas si le workflow d'erreur (Sentinel / Error Workflow) est branché : cette info vit dans les réglages, pas dans le listing minimal. Ouvrir chaque workflow et lire le champ `errorWorkflow`. Déléguer ces lectures lourdes à un sous-agent qui ne renvoie qu'un verdict compact (OK / manquant / autre). Vérifier aussi qu'il n'y a pas d'exécutions en erreur récentes.
3. **Test E2E de l'étage terminal.** Choisir un terrain de test stable. Si les entités sont gelées (statut final), ne pas chercher à rejouer toute la chaîne à froid : exercer l'étage terminal avec les artefacts amont déjà présents. Poser la fixture, actionner le seul déclencheur pertinent (souvent la précondition globale), attendre un cycle, vérifier le résultat, puis remettre la fixture à zéro. Refermer la condition de déclenchement AVANT de nettoyer les objets produits, sinon un runner à scan les reproduit.
4. **Scellage documentaire.** Addendum de clôture (état final : liste des briques, IDs, cadences, filet d'erreur, résultat E2E). POS exact + POS générique. Cocher la milestone. Mettre à jour la mémoire et la passation. Triple save.

## Difficultés typiques

- Le listing d'un gros parc dépasse la limite de contexte.
- L'état `active` ne renseigne pas le branchement du workflow d'erreur.
- Le terrain de test est gelé : pas de cascade complète à froid sans toucher au garde-fou de statut.
- Un runner à scan re-remplit ce qu'on vient de vider si la condition de déclenchement est encore vraie.

## Solutions

- Sauvegarder la sortie volumineuse en fichier, filtrer par IDs connus.
- Lire les réglages (pas seulement l'état) ; déléguer les lectures lourdes à un sous-agent, rendu compact.
- Recentrer l'E2E sur l'étage terminal avec artefacts amont en place ; un déclencheur bien choisi peut prouver le comportement multi-items.
- Imposer un ordre de reset : casser la condition de déclenchement, puis restaurer les objets.

## Lessons

- Vérifier, ne pas supposer : « le filet est branché partout » se lit dans les réglages de chaque workflow.
- La preuve E2E honnête sur un terrain gelé exerce le dernier maillon, elle ne rejoue pas toute la chaîne.
- Sur un runner à scan, l'ordre des écritures de reset fait partie de la correction.
- Déléguer les relevés lourds à un sous-agent préserve le contexte de la session principale pour le raisonnement.
- La clôture n'est réelle qu'une fois le scellage documentaire fait (addendum + POS + spec cochée + mémoire + passation + triple save).
