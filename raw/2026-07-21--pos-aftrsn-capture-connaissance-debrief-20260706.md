---
type: raw
title: "POS-AFTRSN_Capture-Connaissance-Debrief_20260706"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_capture-connaissance-debrief_20260706.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-AFTRSN — Capture de connaissance post-event (débrief → mémoire insights)

**Portée :** AFTRSN. Snapshot daté d'un état vérifié (W-7 publié 06.07.2026). Paire générique : `POS-GENERIQUE_Capture-Connaissance-Debrief_20260706.md`.

---

## 1 · But

Transformer chaque édition terminée en connaissance réutilisable, stockée et traçable dans `aftrsn_memory` collection **`insights`**, avec gate humain.

## 2 · Modèle deux temps (Schedule quotidien, pas de webhook)

**Phase A — Ouvrir (auto sur `Done`).** Éditions `Status=Done` sans fiche débrief → agent pré-remplit (AI-Router `draft`) → fiche créée dans base **Post-Event Débriefs** (`21f26092-…`), reliée à l'édition, `À remplir`, canevas qualitatif → Telegram. Idempotence = existence de fiche.

**Phase B — Capturer (sur `Prêt`).** Karter complète la fiche et la passe en `Prêt` → lecture du corps → agent extrait 3–7 insights JSON (AI-Router `extract`) → écriture dans `insights` via LUMINA-MEMORY-WRITE (`zu4jfZbmDz8trQLl`) → fiche `Capturé` + `Capturé le` + Telegram + épisode. Idempotence = statut `Prêt`→`Capturé`.

## 3 · Traçabilité (invariant mémoire)

Chaque insight écrit avec : `collection=insights`, `knowledge_type=insight`, `source=knowledge-capture`, `source_ref=<nom édition>`, `content_hash=md5(content)` (dédup automatique). Embedding figé `text-embedding-3-small` (1536 dims). Lien Notion↔mémoire par le nom d'édition.

## 4 · Points d'attention AFTRSN

- Base cible = **Post-Event Débriefs** (créée pour W-7), pas de doublon.
- Filtre Notion `Status=Done` en JSON littéral : `={"filter":{"property":"Status","select":{"equals":"Done"}}}`.
- Écriture mémoire via **Execute Workflow** (jamais HTTP direct vers pgvector).
- Synergie : W-6 Budget-Tracker clôt l'édition en `Done` à J+3 → W-7 ouvre le débrief le lendemain matin. Aucun couplage direct (chacun poll Notion).

## Difficultés rencontrées
- Interaction « auto sur Done » + « fiche dédiée » à clarifier → tranché en modèle deux temps.
- Écriture mémoire KO le 05.07 (panne credential Postgres, hors W-7).

## Solutions implémentées
- Deux branches parallèles sur un Schedule quotidien ; base Notion dédiée avec statut pilote ; gate humain sur `Prêt`.
- Panne infra traitée séparément (voir POS-LUMINA credential Postgres).

## Leçons apprises
- **Le gate humain vit dans un statut Notion** (`Prêt`) : simple, visible, idempotent.
- Un pré-remplissage IA factuel + un canevas fixe fiable = meilleure combinaison (l'IA rédige le résumé, la structure reste déterministe).
- Toujours distinguer un bug de workflow d'une panne d'infra partagée avant de « corriger » le workflow.

---
*POS rédigée le 06.07.2026 (W-7 publié).*
