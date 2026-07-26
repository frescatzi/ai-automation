---
type: wiki
title: "AFTRSN — Architecture par processus métier"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [AFTRSN/raw/2026-07-21--aftrsn-architecture-processus-20260704.md]
related: []
updated: 2026-07-21
---

# AFTRSN — Architecture par processus métier

Cadre retenu par Karter (04.07.2026) pour construire l'automatisation AFTRSN : on découpe par **processus métier**, pas par workflow technique. Chaque processus est simple, efficace, lisible seul sur le long terme ; les optimisations viendront de l'usage réel, pas d'une restructuration théorique a priori. On repart de l'existant (retirer l'obsolète, garder le fonctionnel), jamais d'une reconstruction à zéro.

Ce document remplace l'ancien découpage technique W-1a→W-1g (qui découpait le workflow, pas le métier).

## La chaîne métier de référence

Cycle de vie d'une édition, du déclencheur à la clôture :

1. **P1 — Création d'édition** : automatique, sans validation humaine, dès qu'une date est confirmée par le lieu.
2. **P2 — Pack promo** : humain (Canva + core team).
3. **P3 — Publicités** : auto-brouillon + revue humaine, exécution via Hermès.
4. **P4 — Site web & Newsletter** : auto-construction, publication manuelle.
5. **P5 — Post-event** : médias, debrief, closing budget.

Un processus = un workflow n8n (ou une activité humaine outillée). Pas de sous-découpage technique tant que la réalité n'en montre pas le besoin.

## Chaîne d'exécution : Maestro délègue, Hermès exécute

Décision Karter (04.07.2026) : **Maestro orchestre et délègue, il n'exécute jamais rien lui-même.** Toute action automatique sur un outil (Drive, Notion, Calendar, Wix, plateformes pub…) passe par **Hermès**, la plateforme d'exécution partagée (passerelle n8n `LUMINA-Hermes-Exec`). La gouvernance est intégrée au code des skills d'Hermès :
- `[autonomous]` → exécute le non-critique.
- `[supervised]` → propose seulement.
- Action critique (dépense, envoi, publication, suppression) → validation Karter, sans exception.

Mode d'exécution retenu (option b) : Hermès pilote directement les outils avec ses propres skills/runbooks, sous supervision de Maestro. Les workflows n8n existants (ex. `AFTRSN-Event-Creator`) servent de référence de migration vers la bibliothèque Skills d'Hermès.

### Cycle de supervision Maestro ↔ Hermès

1. **Délégation** — Maestro confie la tâche à Hermès avec objectif + **critères d'acceptation** explicites (sans eux, le contrôle en 4 reste subjectif).
2. **Exécution** — Hermès sélectionne ses skills et exécute selon la gouvernance ci-dessus.
3. **Rapport** — Hermès rend le travail + le rapport d'exécution (résultat, skills utilisés), déjà journalisé par `LUMINA-Hermes-Exec` (`Build Log → Log Skills`).
4. **Contrôle** — Maestro (ou le spécialiste concerné) vérifie le rendu contre les critères et la cohérence avec les banques de données ; toute connaissance manquante est complétée.
5. **Issue** — Accepté : Maestro écrit dans la banque de données ce qui a été fait et avec quels skills (visible par tous les agents). Rejeté : diagnostic connaissance vs skill, la cause est notée même en cas d'échec (nourrit la boucle) ; arbitrage Karter au-delà d'un aller-retour pour éviter les boucles infinies entre agents.
6. **Amélioration continue** — les agents poussent leurs connaissances/skills vers Hermès ; la bibliothèque Skills s'enrichit des succès comme des rejets.

## Détail par processus

### P1 — Création d'édition
Une date reçue du lieu déclenche automatiquement tout l'administratif, identique à chaque fois, sans validation humaine. Workflow `AFTRSN-Event-Creator`, conservé tel quel : upsert Venue → fiche Edition (hub central) → checklist → arborescence Drive → événement Calendar → lignes budget → épisode mémoire. Le budget est en auto-brouillon (lignes pré-remplies à terme avec moyennes historiques + capacité du lieu), finalisation toujours humaine.

**État : en production (validé le 04.07.2026).** Découpé en un orchestrateur (9 nodes) + 6 sous-workflows métier (validation & lieu, dossiers média, fiche edition, calendrier, budget, clôture & mémoire), rangés sous `AFTRSN-PROCESS-01-Event-Creation`. Chaque sous-workflow ne reçoit que ses propres paramètres (plus de référence croisée `$('Node')` entre blocs). Test end-to-end validé (payload jetable), données de test purgées, workflows publiés en tag PROD.

Cible : Maestro appelle aujourd'hui ce workflow directement ; à terme Maestro → Hermès → exécution directe des outils. La logique validée (upsert, arborescence, checklist, contrats de données) devient un runbook de la bibliothèque Skills d'Hermès.

### P2 — Pack promo
Flyer + déclinaisons : travail humain sur Canva, validé par la core team, diffusé manuellement — jamais automatisé. Automatisation future limitée au support (rappels d'échéance, suivi d'état dans la checklist). **État : processus humain, rien à construire pour l'instant.**

### P3 — Publicités
Campagnes auto-créées avec paramètres quasi identiques (seule la ville change, Zürich = référence). Jamais publiées sans revue humaine. Une fois live : monitoring + propositions d'optimisation par un agent via Hermès, exécution uniquement après validation Karter. **État : à construire.** Prérequis : runbook « optimisation publicitaire » dans la bibliothèque Skills d'Hermès.

### P4 — Site web & Newsletter
Page événement Wix auto-construite, publication manuelle (bascule auto possible plus tard). Newsletter déclenchée à la publication du site, uniquement si un segment de contacts existe pour la ville. Segmentation aujourd'hui manuelle dans Wix ; son automatisation est un futur workflow séparé, hors périmètre actuel. **État : à construire.**

### P5 — Post-event
Remplissage du dossier photos & vidéos (créé en coquille vide par P1, usage post-promo), debrief, capture de connaissance, closing budgétaire. Repose sur le futur workflow Knowledge-Capture (ingestion debriefs → `aftrsn_memory`, collection `insights`) et le volet closing du suivi budget. **État : à construire.**

## Processus transverses (hors chaîne, en soutien)

- **Coordination lieux** : outreach venues, follow-ups J+3/J+7 → aboutit à la date confirmée qui déclenche P1. À construire.
- **Coordination artistes** : outreach booking, follow-ups, brief D-7. À construire.
- **Réponse email** : lecture Gmail horaire → drafts → notification Telegram détaillée → mémoire ; jamais d'envoi automatique. **En production depuis le 04.07.2026.**
- **Suivi budget** : lecture continue, alertes seuils, rapport de closing (utilisé par P5). À construire.
- **Exécution agents (Hermès)** : plateforme partagée, monitoring/optimisation pubs (P3) ; toute dépense/envoi/publication/suppression passe par validation Karter. Existant.

## Règles de construction (tous processus)

1. Vue simple et limitée par étape : un gros processus se découpe en sous-workflows métier appelés par un orchestrateur — chaque sous-workflow ne reçoit que ses propres paramètres, jamais de référence croisée `$('Node')` entre blocs.
2. Maestro délègue, Hermès exécute — toute action automatique passe par Hermès, quel que soit l'outil piloté.
3. Toute numérotation commence à 1, jamais à 0 (règle permanente).
4. Toute action critique (dépense, envoi, publication, suppression) est gatée par validation humaine — l'administratif de P1 est la seule exception, voulue et bornée aux actions non critiques.
5. Chaque run significatif journalise un épisode dans `aftrsn_memory` via `LUMINA-MEMORY-WRITE/WEBHOOK`.
6. Convention n8n : un processus = un dossier `AFTRSN-PROCESS-<nn>-<Nom>` sous `AFTRSN-02-BUSINESS AUTOMATION`, contenant l'orchestrateur et ses sous-workflows ; l'infrastructure vit sous `AFTRSN-01-INFRASTRUCTURE`. Payload jetable `ZZTEST-…` avant toute donnée réelle.

## Invariants repris de l'ancien découpage

- Périmètre métier : administratif automatique en P1 ; promo/pubs/site/newsletter en aval, gatés humain.
- Gestion d'erreur : pas d'écriture partielle silencieuse, aucune action destructive, sortie d'échec `{status:"error", step, message}`.
- Filtre Notion « Contains » (jamais « Equals » sur un titre), PATCH pour l'injection checklist, HTTP Request pour Claude, mémoire épisodique à chaque run significatif.
- DB cibles 🎭 : Venues, Editions, Budget & Finance (identifiants dans le raw source).

Obsolète et retiré : le schéma technique W-1a→W-1g et son orchestrateur (découpait le workflow, pas le métier) ; la dépendance `folder_id` → calendrier (le calendrier pointe désormais vers la fiche Notion, hub central).
