---
type: raw
title: "POS-EXACT_AutoEvent-P3_M9-verification-finale-cloture_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m9-verification-finale-cloture_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · M9 (Vérification finale + Clôture)

Date : 20.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Moteur : n8n LUMINA OS × Notion « The Backstage »
Milestone : M9 (dernière milestone de la Phase 3). Type : vérification et scellage, aucune brique nouvelle.

## Ce qui a été fait (déroulé exact)

1. **Santé + inventaire par API.** `n8n_health_check` (instance 2.65.1, OK). `n8n_list_workflows` : sortie trop volumineuse (parc LUMINA entier, ~2000 lignes) renvoyée dans un fichier. Grep ciblé sur les 13 IDs de la Phase 3 : les 12 briques + le Sentinel sont tous `active: true`.
2. **Audit Sentinel (délégué à un sous-agent, lecture seule).** Pour chacun des 12 workflows, lecture de `settings.errorWorkflow`. Résultat : 12/12 pointent sur `xzaH0uWy0idKVphF`. Revue des exécutions filtrées `status=error` : 0 erreur.
3. **Test final E2E (convergence M8b sur EP05).** Relevé de la fixture par requêtes Notion SQL sur les 3 data sources Content + les 2 tâches Go-live. Newsletter (Announce, Line-up reveal) = `Sent` ; Instagram = `Posted` ; Website `EP05 - AFTRSN - HOME · Event page` = `To Validate` ; tâches Go-live = `To Do`, Notes vides. Flip manuel de la seule ligne Event page HOME → `Approved`. Attente d'un cycle de scan (< 2 min). Relecture : les 2 tâches Go-live remplies (récap prêt à diffuser) et passées `To Validate`. Remise en fixture (Event page → `To Validate`, tâches → `To Do`, Notes vidées), vérifiée par requête.
4. **Doc de clôture.** Addendum Bible v3.19, POS ×2, triple save, spec cochée, mémoire, passation finale. Bible consolidée réécrite (livrable séparé).

## Difficultés

- `n8n_list_workflows` dépasse la limite de tokens : le parc n8n complet ne se lit pas d'un bloc.
- `settings.errorWorkflow` n'est pas exposé par le listing minimal : il faut ouvrir chaque workflow en `full` (payload lourd, ×12).
- EP05 est `Done` donc gelée : impossible de dérouler un vrai M1 → M8b « à froid » sans toucher au `Status` de l'édition (zone sensible).
- Risque de re-remplissage : si on vide les tâches Go-live alors que l'Event page est encore `Approved`, le scan M8b les re-remplit au cycle suivant.

## Solutions

- Sortie du listing sauvegardée en fichier, puis grep ciblé sur les 13 IDs connus (pas de lecture intégrale).
- Audit des 12 `settings.errorWorkflow` délégué à un sous-agent qui ne renvoie qu'un tableau compact (contexte principal préservé).
- E2E recentré sur l'étage terminal (convergence M8b), avec les artefacts amont déjà présents sur EP05 : un flip unique de l'Event page prouve la double convergence (Announce + Line-up reveal) en une passe.
- Ordre de reset imposé : refermer d'abord la précondition (Event page → `To Validate`), puis vider les tâches. La convergence étant fausse, le scan ne re-remplit pas.

## Lessons

- Audit d'état sur un gros parc n8n : filtrer par IDs connus vaut mieux que lister tout ; déléguer les lectures lourdes (settings) à un sous-agent garde le contexte propre.
- La preuve E2E la plus honnête sur un terrain gelé consiste à exercer l'étage terminal avec les artefacts amont déjà en place, pas à rejouer toute la chaîne depuis M1 (ce qui exigerait de dégeler une édition).
- Sur un runner à scan, l'ordre des écritures de reset compte : casser la condition de déclenchement avant de remettre l'objet muté à l'état initial.
- Un seul déclencheur bien choisi (la précondition globale) peut valider le comportement multi-items (plusieurs vagues convergent ensemble).
- « Sentinel partout » se vérifie, ne se suppose pas : le listing dit `active`, pas `errorWorkflow`. La vérification finale doit lire les réglages, pas seulement l'état actif.
