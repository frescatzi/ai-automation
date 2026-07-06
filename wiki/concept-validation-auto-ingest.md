---
type: wiki
title: "Concept — Validation de l'auto-ingest (raw → wiki → Git)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/raw/2026-07-06--test-auto-ingest.md
  - raw/raw/2026-07-06--test-auto-ingest-2.md
  - raw/2026-07-06--test-auto-ingest-2.md
  - raw/2026-07-06--test-3.md
related:
  - concept-pipeline-memoire-wiki-git
  - concept-intake-source-git
  - concept-archivage-n8n-idempotent
updated: 2026-07-06
---

# Concept — Validation de l'auto-ingest (raw → wiki → Git)

> **Origine :** cette page est née d'une note de test (`raw/raw/2026-07-06--test-auto-ingest.md`) déposée pour valider le runner d'auto-ingest **de bout en bout**. Une seconde note (`raw/raw/2026-07-06--test-auto-ingest-2.md`) a confirmé le **déclenchement automatique** : un push GitHub réveille n8n → runner → compilation → push, sans intervention. Elle est en `draft` et peut être retirée après revue.

---

## Ce que la note validait

Le test vérifie que la chaîne **auto-ingest** fonctionne sans intervention manuelle :

1. Une source arrive dans `raw/`.
2. Le runner (LLM-bibliothécaire) la **compile** en une page `wiki/` propre et back-linkée.
3. Le résultat est **poussé sur `main`** (le service git s'en charge, pas le LLM).

La production de *cette* page `wiki/` — reliée au wiki et journalisée — est donc le **signal de succès** du test.

Le second test (`test-auto-ingest-2`) ajoute une garantie sur le **déclencheur** lui-même : ce n'est pas seulement la compilation qui marche, c'est le fait qu'un simple `push` sur `main` suffit à démarrer toute la chaîne (webhook GitHub → n8n → runner). Le corps des deux notes diffère légèrement mais elles valident le **même maillon** — d'où une page unique enrichie plutôt qu'un doublon. Une **re-capture** de `test-auto-ingest-2` est ensuite arrivée à la racine `raw/` (`raw/2026-07-06--test-auto-ingest-2.md`), au corps **identique** à la copie déjà compilée sous `raw/raw/` : elle a simplement été ajoutée aux `sources:` (provenance préservée) et mise en file d'archivage, sans page nouvelle — application directe de la règle de dédup.

Une troisième note (`test-3`) reformule le même test de déclencheur (« valider que le push GitHub déclenche automatiquement n8n → runner → compilation → push ») — même maillon que `test-auto-ingest-2`, donc simple ajout aux `sources:` plutôt qu'une page distincte.

## Rattachement au pipeline

Ce test valide concrètement le maillon amont du pipeline décrit dans
[[concept-pipeline-memoire-wiki-git]] : Git reste la source unique, et les vues dérivées
(base vectorielle des agents, outil de doc des humains) se régénèrent depuis ce wiki.
Le drain idempotent de la source suit le pattern [[concept-intake-source-git]], et
l'archivage post-compilation suit [[concept-archivage-n8n-idempotent]].

## Limites / conditions d'usage

- **Aucune connaissance métier réutilisable** ici : c'est un artefact de validation. La note source précise « À supprimer ensuite ».
- La page reste en `status: draft` / `publish: none` : promotion = décision humaine.
- Les fichiers sources sont **immuables** ; ils seront drainés par le service d'archivage (queue), pas supprimés par le LLM.
