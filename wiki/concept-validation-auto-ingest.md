---
type: wiki
title: "Concept — Validation de l'auto-ingest (raw → wiki → Git)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/raw/2026-07-06--test-auto-ingest.md
related:
  - concept-pipeline-memoire-wiki-git
  - concept-intake-source-git
  - concept-archivage-n8n-idempotent
updated: 2026-07-06
---

# Concept — Validation de l'auto-ingest (raw → wiki → Git)

> **Origine :** cette page est née d'une note de test (`raw/raw/2026-07-06--test-auto-ingest.md`) déposée pour valider le runner d'auto-ingest **de bout en bout**. Elle est en `draft` et peut être retirée après revue.

---

## Ce que la note validait

Le test vérifie que la chaîne **auto-ingest** fonctionne sans intervention manuelle :

1. Une source arrive dans `raw/`.
2. Le runner (LLM-bibliothécaire) la **compile** en une page `wiki/` propre et back-linkée.
3. Le résultat est **poussé sur `main`** (le service git s'en charge, pas le LLM).

La production de *cette* page `wiki/` — reliée au wiki et journalisée — est donc le **signal de succès** du test.

## Rattachement au pipeline

Ce test valide concrètement le maillon amont du pipeline décrit dans
[[concept-pipeline-memoire-wiki-git]] : Git reste la source unique, et les vues dérivées
(base vectorielle des agents, outil de doc des humains) se régénèrent depuis ce wiki.
Le drain idempotent de la source suit le pattern [[concept-intake-source-git]], et
l'archivage post-compilation suit [[concept-archivage-n8n-idempotent]].

## Limites / conditions d'usage

- **Aucune connaissance métier réutilisable** ici : c'est un artefact de validation. La note source précise « À supprimer ensuite ».
- La page reste en `status: draft` / `publish: none` : promotion = décision humaine.
- Le fichier source est **immuable** ; il sera drainé par le service d'archivage (queue), pas supprimé par le LLM.
