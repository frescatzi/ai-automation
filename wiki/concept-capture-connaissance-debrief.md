---
type: wiki
title: "Concept — Capture de connaissance post-projet (débrief → mémoire)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-07-06--pos-generique-capture-connaissance-debrief-20260706.md"
  - "raw/2026-07-07--pos-generique-capture-connaissance-debrief-20260706.md"
related:
  - "concept-archivage-n8n-idempotent"
  - "concept-pipeline-memoire-wiki-git"
  - "concept-memoire-vivante-agents"
  - "sop/sop-creer-memoire-agents-humains"
updated: 2026-07-13
---

# Concept — Capture de connaissance post-projet (débrief → mémoire)

> **Idée centrale :** transformer chaque unité de travail terminée (édition, projet, campagne, sprint) en connaissance réutilisable et traçable, stockée dans la mémoire de la marque (`collection=insights`), **avec validation humaine**. Complète la boucle CONSOLIDATE de la [[concept-memoire-vivante-agents]] par une capture **volontaire et qualitative**, plutôt que purement automatique.

*(Note de provenance : deux exports identiques du même patron, capturés le 2026-07-06 et le 2026-07-07 — même contenu, pas de divergence à réconcilier.)*

## Modèle deux temps (schedule périodique)

**Phase A — Ouvrir (auto, sur statut « terminé »).**
Les entités passées au statut terminal sans fiche débrief → un agent rédacteur pré-remplit un résumé factuel depuis les données déjà connues → création d'une **fiche débrief dédiée**, reliée à l'entité, statut `À remplir`, avec un **canevas qualitatif fixe** (ce qui a marché / raté / chiffres réels / à refaire / à éviter / idées). Notification. **Idempotence = existence de fiche.**

**Phase B — Capturer (sur statut « prêt »).**
L'humain complète la fiche et la passe en `Prêt` → un agent lit le corps et extrait N insights structurés (JSON strict, autonomes, actionnables, tagués par thème) → écriture de chaque insight dans la mémoire → fiche `Capturé` + horodatage + notification. **Idempotence = transition `Prêt` → `Capturé`.**

## Traçabilité (invariant)

Chaque insight porte : collection dédiée, `knowledge_type`, `source`, `source_ref` (= identifiant de l'entité d'origine), `content_hash` (dédup), embedding figé. Le lien entité ↔ mémoire passe toujours par un identifiant stable.

## Règles de robustesse

- **Gate humain via un statut** dans l'outil de gestion déjà utilisé — pas de capture avant validation.
- **Pré-remplissage IA factuel + canevas fixe déterministe** : l'IA résume, elle ne fabrique jamais la structure.
- **Idempotence à double garde** (existence de fiche en Phase A, transition de statut en Phase B) → un poll périodique ne duplique jamais.
- **Écriture mémoire via le sous-workflow d'écriture dédié**, jamais d'insertion SQL directe depuis le workflow métier (cf. primitive WRITE de [[concept-memoire-vivante-agents]]).
- **Parser d'insights défensif** : retirer les fences markdown, isoler le tableau JSON, tolérer 0 insight (marquer quand même `Capturé` pour ne pas retraiter indéfiniment).

## Difficultés, solutions, leçons

| Difficulté | Solution retenue |
|---|---|
| Ambiguïté entre « déclenchement automatique » et « fiche dédiée à remplir » | Le modèle **deux temps** (Ouvrir auto / Capturer sur validation) lève l'ambiguïté : l'automatisation ouvre, l'humain valide. |
| Dépendance à une infra mémoire partagée (une panne de credential bloque l'écriture) | Traiter les pannes d'infra partagée dans un **runbook séparé**, pas dans la logique métier du débrief. |

**Leçons apprises :**
1. Le meilleur endroit pour un gate humain récurrent est un **champ statut** déjà visible dans l'outil que l'humain utilise au quotidien — pas un outil séparé.
2. Séparer nettement **rédaction IA** (résumé, insights) et **structure déterministe** (canevas, blocs fixes) donne à la fois fiabilité et intelligence.
3. Avant de corriger un workflow qui échoue en écriture mémoire, **vérifier l'infra partagée** (credential, base) avant de suspecter la logique métier.

## Voir aussi

- [[concept-memoire-vivante-agents]] — la mémoire cible (`collection=insights`) et sa primitive WRITE idempotente.
- [[concept-pipeline-memoire-wiki-git]] — pendant « connaissance versionnée en wiki » de cette capture « connaissance en base mémoire ».
- [[concept-archivage-n8n-idempotent]] — même famille de patron (manifeste/statut pilote + garde-fous d'idempotence).
- [[sop/sop-creer-memoire-agents-humains]] — implémentation concrète du sous-workflow d'écriture mémoire utilisé en Phase B.
