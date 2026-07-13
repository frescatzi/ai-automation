---
type: wiki
title: "SOP — Apprendre un skill à Hermès (écriture directe dans la bibliothèque)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-07-07--pos-lumina-apprendre-un-skill-a-hermes-20260705.md"
related:
  - "concept-bibliotheque-skills-apprenante"
  - "concept-hermes-agent"
  - "concept-memoire-vivante-agents"
  - "sop/sop-creer-memoire-agents-humains"
updated: 2026-07-13
---

# SOP — Apprendre un skill à Hermès (écriture directe dans la bibliothèque)

> Procédure vérifiée le 2026-07-05 — premier skill appris à Hermès sur décision Karter (« Hermès est l'exécutant et l'apprenant ; les spécialistes sont là pour le guider ») : skill n°990 « Recherche d'historique email envoyé (Gmail) ». Générique par nature (architecture LUMINA OS) — pas de version spécifique par marque.

## Principe

La capacité d'[[concept-hermes-agent]] n'est pas définie par ses nodes n8n mais par sa **bibliothèque de skills** ([[concept-bibliotheque-skills-apprenante]]) : Hermès embed la tâche reçue, cherche les skills pertinents par similarité (Postgres/pgvector), exécute, puis journalise. **Apprendre un skill à Hermès = écrire un runbook dans la table mémoire, `collection=skills`** — il le trouvera seul dès que la tâche s'y prête.

## Procédure

1. **Rédiger le runbook** au format des skills existants (regarder un skill en place via `search_brand_memory` avant d'écrire) : `RUNBOOK:` (une ligne de définition) / `BUT:` (usages) / `ETAPES:` numérotées à partir de 1 / section `AMELIORATION CONTINUE:` (ce qu'Hermès doit consigner à chaque usage) / `REGLE:` (invariants, ex. lecture seule, jamais d'envoi) / un **cas d'origine daté** comme exemple concret. Référencer les outils d'exécution concrets (ex. workflow utilitaire n8n avec son ID).
2. **Écrire via le sous-workflow d'écriture mémoire dédié** (`LUMINA-MEMORY-WRITE/WEBHOOK`, id `zu4jfZbmDz8trQLl`) — malgré son nom, son seul trigger est `Execute Workflow` (**pas** de webhook réel) : il faut un **appelant** (node Execute Workflow) avec le contrat exact `{brand, title, content, collection:"skills", knowledge_type:"skill", source, source_ref}`. `brand:"lumina"` pour un skill partagé entre marques, `"aftrsn"` pour du pur métier de marque. L'appelant est jetable → nommage `ZZZ_` + décision de suppression pour le fondateur.
3. **Vérifier la sortie** : le write renvoie `{ok:true, table, collection}` — contrôler que `collection:"skills"`.
4. **Vérifier la trouvabilité (étape décisive)** : interroger `search_brand_memory` avec une **requête formulée comme une vraie tâche** (pas comme le titre du skill). Le skill doit sortir en tête, avec une similarité nettement au-dessus des voisins — c'est ce test qui prédit qu'Hermès le sélectionnera en usage réel.

## Difficultés rencontrées → solutions implémentées

| Difficulté | Solution |
|---|---|
| `LUMINA-MEMORY-WRITE/WEBHOOK` n'expose pas de webhook malgré son nom — impossible de POSTer directement | Appelant jetable `ZZZ_` avec node Execute Workflow, contrat `jsonExample` reproduit exactement ; suppression déléguée au fondateur |
| Taxonomie du contrat incertaine (`knowledge_type` pour un skill ? quelle `brand` ?) — aucun schéma documenté | Alignement sur l'existant : symétrie `episodic`/`episode` → `skills`/`skill` ; `brand` déduite de l'emplacement des skills frères (les runbooks d'outreach vivent dans `lumina`) |
| Risque de skill « invisible » : un runbook mal formulé peut être sémantiquement loin des requêtes réelles d'Hermès | Test de trouvabilité post-écriture avec une requête d'usage réel (« Vérifier si un lieu a déjà été contacté par email et retrouver le fil… ») → sorti 1er à 0.66, devant les skills voisins (≤0.60) |

## Leçons apprises

1. **Enseigner à Hermès = écrire dans la banque, pas câbler du n8n** — le workflow utilitaire référencé par le skill n'est qu'un outil, pas le mécanisme d'apprentissage.
2. Le nom d'un workflow ne décrit pas son trigger : toujours vérifier le node trigger avant de choisir le mode d'invocation.
3. Un contrat non documenté se déduit **des données existantes** (skills frères), pas d'une supposition — et se vérifie par la réponse du write.
4. L'écriture ne suffit pas : un skill n'existe pour Hermès que s'il **remonte en tête d'une recherche formulée comme une tâche**. Le test de trouvabilité fait partie intégrante de la procédure, pas une option.

## Voir aussi

- [[concept-bibliotheque-skills-apprenante]] — le concept général (deux stockages, boucle consulter/journaliser/graduer/créer) dont cette SOP est l'application manuelle « créer ».
- [[concept-hermes-agent]] — l'agent qui consulte, exécute et gradue ce skill une fois écrit.
- [[concept-memoire-vivante-agents]] — la primitive WRITE générique que ce sous-workflow implémente pour `collection=skills`.
- [[sop/sop-creer-memoire-agents-humains]] — construction de la couche mémoire (pgvector) qui héberge la bibliothèque de skills.
