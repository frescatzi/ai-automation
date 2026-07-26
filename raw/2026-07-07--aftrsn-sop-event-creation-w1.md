---
type: raw
title: "AFTRSN_SOP_Event-Creation_W1"
source_url: "drive:1bg3H8FN2SuxTs2AoC0LgNM42BWSg4Toj"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# RUNBOOK — Event Creation (W-1)
**Statut :** Canon v2 — réconcilié avec le Notion réel ("The Backstage") le 02.07.2026
**Lié à :** W-1 · AFTRSN-Event-Creator (spec technique n8n : `W1_Event-Creator_Flowchart_Spec_20260702.md`)
**Usage :** Destiné à la collection `canon` de `aftrsn_memory`. Référence pour Maestro et pour Karter avant de lancer la commande.

> ⚠ Ce runbook parle d'**Edition** (DB Notion réelle), pas d'"Event" — vocabulaire à respecter pour rester cohérent avec le Notion existant (The Backstage → Operations → Editions).

---

## But
Faire exister une Edition AFTER SUN dans tout le système (Notion, Drive, Calendar, amorce budget) en une seule commande, en s'appuyant sur les DB Notion déjà en place plutôt que d'en créer de nouvelles.

## Quand déclencher
- **Idée précoce (`Status = Planning`)** — dès qu'une date et une ville se dessinent, même sans venue confirmée.
- **Confirmation** — dès que la venue est confirmée dans la DB Venues (`Status = Confirmed`), la page Edition bascule en `Status = Confirmed`.

Ne jamais laisser une venue ou un artiste `Confirmed` sans page Edition associée — c'est le signal qu'il faut lancer W-1 immédiatement si ce n'est pas déjà fait.

## Pré-requis
- Nom de l'édition, ville et date (obligatoires)
- Venue (optionnel à ce stade)
- Jauge cible (`Capacity target`, optionnel)
- Credential Google OAuth AFTRSN actif (Drive, Calendar)
- Accès Notion opérationnel (déjà confirmé)

## Étapes (vue métier — détail technique dans la spec n8n)
1. Lancer la commande via Maestro avec édition + ville + date (+ venue si connue).
2. Le système crée en parallèle : dossier Drive structuré (avec sous-dossiers média au format canon `YYYYMMDD_AFTSN_Photos/Videos`, `Raw`/`Edited`), entrée Calendar.
3. Le système écrit la page dans **Editions**, liée à la venue (si fournie) via la DB **Venues**.
4. Le système injecte la checklist standard directement dans le corps de la page (mêmes 7 sections que le template existant : basics, lineup, budget, contenu, marketing/KPI, partenaires, media assets, post-event).
5. Le système amorce des lignes vides dans **Budget & Finance** (Venue, Artists, Production, Marketing, Media) reliées à l'édition.
6. Le système sauvegarde un épisode dans `aftrsn_memory` (collection `episodic`).
7. Karter reçoit un résumé avec les liens directs — à vérifier une fois.

## Checklist injectée (reprend le template Notion existant)
1. Lock the basics — City + Venue confirmés, date verrouillée, capacité, statut → Confirmed
2. Lineup & artists — DJs confirmés, champ Lineup rempli, fees loggés dans Budget & Finance
3. Budget — lignes Venue / Artists / Production / Marketing / Media
4. Content & Social — announcement, lineup reveal, teaser, countdown, recap reel
5. Marketing & KPI — push billetterie/guestlist, objectif reach
6. Partners & Sponsors — partenaire boisson, partenaire média
7. Media Assets — photographe/vidéaste booké, référence hero, LUT, dossiers nommés correctement
8. Post-event — lien recap ajouté, statut → Done

## Points de décision
- **Venue inconnue au moment de la création ?** Créer quand même la page Edition en `Planning` — la relation Venue se rattrape plus tard.
- **Deadline J-21 ratée ?** La checklist ne se corrige pas automatiquement — Karter doit ajuster la fenêtre marketing (impact sur W-4).
- **Plusieurs villes/éditions en parallèle ?** Chaque édition = une page distincte dans **Editions**, jamais partagée entre deux dates ou deux villes.

## Erreurs à éviter
- Créer la page Editions à la main pendant qu'un run W-1 est en cours — casse la correspondance Notion ↔ Drive ↔ Calendar garantie par construction.
- Recréer une DB "Events" ou "Event Tasks" parallèle — **elle n'existe pas et ne doit pas exister**, tout vit dans Editions + le contenu de page.
- Utiliser un vocabulaire de statut inventé (`Idea`, `Confirmed`/`Rejected` pour Venues) au lieu des valeurs réelles (`Planning`/`Prospect`/`In discussion`/`Active`).

## Ce que ça déclenche ensuite
- **W-2** Venue-Coordinator et **W-3** Artist-Coordinator mettent à jour les DB Venues / DJs-Artists reliées à cette édition.
- **W-4** Marketing-Planner se déclenche quand `Status = Confirmed`.
- **W-6** Budget-Tracker lit et enrichit les lignes Budget & Finance créées ici.
- **W-7** Knowledge-Capture, post-event, clôt la boucle sur cette même page (`Status = Done`, `Recap link` rempli).

## Definition of Done
Un run W-1 réussi = page Editions créée avec checklist injectée, Drive et Calendar liés, lignes Budget & Finance amorcées, épisode mémoire sauvegardé. Si un seul élément manque, le run est incomplet.

---
*Rédigé le 02.07.2026, réconcilié le même jour après lecture directe du Notion réel. À affiner après le premier run réel.*
