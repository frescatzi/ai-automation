---
type: raw
title: "Concept_Runbook_Definition-et-Usage-LUMINA"
source_url: "drive:18jWh7x-i6XBh8dw2_m3Ct5wGQDVe9Yxn"
captured: 2026-07-04
vault: brands
brand: AFTRSN
immutable: true
---

# Concept — Runbook (définition et usage dans LUMINA)

Un **runbook** est un mode d'emploi opérationnel : la marche à suivre complète pour exécuter une tâche précise, étape par étape, avec les conditions, les vérifications et les cas d'erreur. Le terme vient de l'exploitation informatique — le classeur que l'équipe de nuit ouvrait pour savoir exactement quoi faire, dans quel ordre, sans improviser.

## 1. Ce qui fait un bon runbook

1. **Déclencheur clair** — quand est-ce qu'on l'exécute (ex. « une date confirmée par le lieu »).
2. **Étapes ordonnées** — quoi faire, dans quel ordre, avec quels outils.
3. **Règles et invariants** — les pièges connus figés en consignes (ex. filtre Notion « Contains », jamais « Equals » sur un titre ; méthode PATCH pour injecter du contenu).
4. **Vérifications** — comment savoir que chaque étape a réussi.
5. **Cas d'erreur** — quoi faire quand ça échoue, sans écriture partielle silencieuse.

## 2. Dans LUMINA : skill = runbook exécutable

La bibliothèque **Skills** d'Hermès est une bibliothèque de runbooks qu'un agent sait lire et exécuter (voir [[POS-GENERIQUE_Bibliotheque-de-skills-apprenante-agents]] pour la mécanique maturité/graduation) :

- **`[supervised]`** — Hermès lit le runbook et propose, n'exécute pas.
- **`[autonomous]`** — Hermès exécute les étapes non critiques sans redemander.
- **Invariant** : action critique (dépense, envoi, publication, suppression) → validation Karter, toujours, quel que soit le niveau.

## 3. Exemple concret — runbook « Création d'édition » (P1 AFTRSN)

Reçois `{date, lieu, ville, capacité}` → cherche le venue dans Notion **avant** d'en créer un → crée l'arborescence Drive au format Année/Mois/Édition (la date groupe, jamais la ville) → crée la fiche Edition (hub central) → injecte la checklist (PATCH) → crée l'événement Calendar (description → fiche Notion) → amorce le budget (brouillon, finalisation humaine) → journalise l'épisode mémoire.

## 4. Workflow n8n vs runbook

Un workflow n8n **testé et validé** contient déjà toute la logique d'un runbook (étapes, règles, gestion d'erreur) — c'est une **référence de migration** : on traduit sa logique en runbook que Hermès exécute lui-même. Même recette, autre cuisinier. Principe retenu (décision Karter 04.07.2026, architecture AFTRSN) : Maestro délègue, Hermès exécute les actions automatiques avec ses runbooks, sous supervision de Maestro (rapport des skills utilisés, contrôle, écriture en banque de données, boucle d'amélioration).

---
*Note de connaissance — 2026-07-04. Source : session AFTRSN Automation, `AFTRSN_Architecture_Processus_20260704.md`.*
