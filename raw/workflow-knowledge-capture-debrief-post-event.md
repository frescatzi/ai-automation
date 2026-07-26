---
type: wiki
title: "AFTRSN — Workflow Knowledge-Capture : débrief post-event vers mémoire insights"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [AFTRSN/raw/2026-07-21--pos-aftrsn-capture-connaissance-debrief-20260706.md]
related: [[architecture-processus-metier]], [[methode-construction-workflow-n8n]]
updated: 2026-07-21
---

# AFTRSN — Workflow Knowledge-Capture : débrief post-event vers mémoire insights

Workflow **W-7** (publié le 06.07.2026), qui réalise le volet « capture de connaissance » du processus **P5 — Post-event** (voir [[architecture-processus-metier]]). But : transformer chaque édition terminée en connaissance réutilisable, stockée et traçable dans `aftrsn_memory` (collection `insights`), avec gate humain — jamais d'écriture mémoire sans validation.

## Modèle en deux temps (Schedule quotidien, pas de webhook)

**Phase A — Ouvrir (auto sur `Done`).** Pour chaque édition `Status=Done` sans fiche débrief encore créée, un agent pré-remplit un brouillon (AI-Router `draft`) et crée une fiche dans la base Notion **Post-Event Débriefs**, reliée à l'édition, statut `À remplir`, canevas qualitatif fixe, puis notifie par Telegram. Idempotence : l'existence de la fiche fait foi (pas de doublon si le run repasse).

**Phase B — Capturer (sur `Prêt`).** Le Karter complète la fiche à la main et la passe en statut `Prêt`. Le workflow lit le corps, en extrait 3–7 insights structurés en JSON (AI-Router `extract`), et les écrit dans la collection `insights` via le sous-workflow `LUMINA-MEMORY-WRITE` (`zu4jfZbmDz8trQLl`), appelé en **Execute Workflow** — jamais en HTTP direct vers pgvector. La fiche passe alors à `Capturé`, avec la date de capture, une notification Telegram et un épisode mémoire. Idempotence : la transition de statut `Prêt` → `Capturé`.

**Le gate humain vit entièrement dans le statut Notion `Prêt`** : simple, visible, idempotent — pas de mécanisme de validation séparé à maintenir.

## Traçabilité (invariant mémoire)

Chaque insight est écrit avec : `collection=insights`, `knowledge_type=insight`, `source=knowledge-capture`, `source_ref=<nom édition>`, `content_hash=md5(content)` pour dédup automatique, embedding figé `text-embedding-3-small` (1536 dims). Le lien Notion ↔ mémoire se fait par le nom de l'édition.

## Points d'attention spécifiques

- Base cible = **Post-Event Débriefs** (créée pour W-7) — vérifier qu'aucun doublon de base n'existe avant d'en créer une nouvelle.
- Filtre Notion `Status=Done` à passer en JSON littéral : `={"filter":{"property":"Status","select":{"equals":"Done"}}}`.
- Synergie sans couplage direct avec le suivi budget (W-6 Budget-Tracker) : W-6 clôt l'édition en `Done` à J+3, W-7 ouvre le débrief le lendemain matin — chacun poll Notion indépendamment, pas d'appel croisé.

## Leçons apprises

- L'articulation entre « auto sur `Done` » et « fiche dédiée » n'était pas évidente au départ (risque de double déclenchement ou de fiche manquante) : tranchée par le modèle à deux phases ci-dessus, chacune avec sa propre condition d'idempotence.
- Un pré-remplissage IA factuel combiné à un canevas fixe et déterministe est plus fiable qu'un remplissage entièrement libre : l'IA rédige le résumé, la structure ne varie jamais.
- Toujours distinguer un bug de workflow d'une panne d'infrastructure partagée avant de « corriger » le workflow — une panne du credential Postgres le 05.07.2026 avait bloqué l'écriture mémoire indépendamment de toute logique W-7 (voir POS-LUMINA credential Postgres).
