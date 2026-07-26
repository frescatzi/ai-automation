---
type: raw
title: "Fiche_AFTRSN-Event-Creator_20260703"
source_url: "drive:1wnQf5uBmOUDRMmR9uAfZDvNqyd1rOcIw"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche workflow — AFTRSN-Event-Creator (W-1)

**Format :** conforme à la Bible 360° LUMINA OS §E (fiches détaillées par workflow). Vient compléter l'Addendum v1.1 de la Bible, qui référençait ce workflow comme 33ᵉ de l'inventaire, "à la racine (hors standard : à classer + taguer)".

---

**ID :** `kjuj3r0QVP6ZicIr` — **Criticité :** 🟠 (important — la brique administrative de création d'événement, pas encore dans la chaîne critique 🔴 memory/router/Maestro)
**Statut au 03.07.2026 :** ❌ non publié → **classé et taggué ce jour** (voir §Classement)
**Emplacement :** `Personal / 02-AFTRSN / AFTRSN-02-BUSINESS AUTOMATION / AFTRSN-PROCESS-01-Event-Creation / AFTRSN-Event-Creator` (rangement Karter du 04.07.2026 ; anciennement `AFTRSN-04-AUTOMATION`)
**⚠️ Restructuration majeure le 04.07.2026 :** ce workflow est devenu un **orchestrateur** de 9 nodes appelant 6 sous-workflows métier (AFTRSN-P1.1 → P1.6, mêmes dossier et tags). La description node-par-node ci-dessous reflète l'état monolithique du 03.07 — voir `AFTRSN_Architecture_Processus_20260704.md` pour l'architecture actuelle.
**Tags appliqués :** 🌞 AFTRSN · 🔄 automation · 🔵 TEST *(pas de tag déclencheur — sous-workflow appelé, conforme à la convention Bible §C)*
**Déclencheur :** `When Executed by Another Workflow` — appelé par AFTRSN-Maestro (ou directement, testé manuellement pendant cette session)

## Rôle

Automatise la création d'une nouvelle édition AFTER SUN : reçoit `{edition_name, city, event_date, venue_name, capacity_target, requested_by}`, upsert le lieu (cherche avant de créer), crée l'arborescence Drive, crée la fiche Edition dans Notion, y injecte la checklist standard, crée l'événement Google Calendar, amorce les lignes Budget & Finance, puis journalise l'épisode.

Process métier de référence (business logic, clarifié avec Karter le 03.07.2026) : une date reçue du lieu = un événement, déclenchement automatique complet sans validation humaine pour la partie administrative (dossiers, checklist, fiche, mémoire). Voir `W1_Decoupage_Sous-Workflows_20260703.md` pour la réflexion en cours sur une possible décomposition en sous-workflows.

## Nodes (chaîne complète)

**Entrée & validation**
`When Executed by Another Workflow` → `If` (validation champs requis) → `Edit Fields` (normalisation) / `Erreur - reponse` (branche erreur, manuel)

**Branche Dossiers Média (parallèle)**
`Create folder` (racine, `create:folder`) → `Create folder1` → `Create folder - Photos-Raw` / `Create folder - Photos-Edited` / `Create folder - Videos` → `Create folder - Videos-Raw` / `Create folder - Videos-Edited` → `Merge - Dossiers média` (append, 4 inputs)
⚠️ Cette branche est actuellement **un cul-de-sac** : `Merge - Dossiers média` n'a aucune connexion sortante vers le reste du workflow (confirmé par inspection directe des edges le 03.07.2026). Elle tourne mais son résultat n'est utilisé nulle part — notamment pas par `Create an event`, qui devrait pourtant y faire référence pour le lien du dossier média (cf. §Travaux du jour).

**Branche Lieu — upsert (chercher avant créer)**
`Notion - Chercher Venue` (`getAll:databasePage`, DB Venues, filtre **Contains** sur le titre) → `Verifier Venue existante` (Code, *Always Output Data* activé) → `If1` → branche vraie : `Edit Fields1` (récupère `venue_id` existant) ; branche fausse : `Create a database page` (crée le Venue) → `Venue - Nouvelle` (récupère le nouvel id) → les deux convergent dans `Merge1` (append — sûr ici car les branches sont mutuellement exclusives)

**Création de l'édition**
`Merge1` → `Create a database page1` (`create:databasePage`, DB Editions — Venue en relation vers `venue_id`)

**Post-création (parallèle)**
- `Create a database page1` → `Create an event` (Google Calendar, `create:event`) — ⚠️ voir §Travaux du jour, en pause
- `Create a database page1` → `Notion - Injecter checklist` (HTTP Request → Notion API, méthode **PATCH**) → `Preparer lignes Budget` (Code) → `Notion - Amorcer Budget & Finance` (`create:databasePage`) → `Merge2` (append, ajouté le 03.07.2026, pas encore câblé en sortie)

**Clôture**
`Create an event` + branche budget → `Merge` (append) → `Memoire - Save Episode` (executeWorkflow → `LUMINA-MEMORY-WRITE/WEBHOOK`) → `Agreger resume` (Set, manuel)

## Dépendances

- **Amont :** AFTRSN-Maestro (appelant), DB Notion Venues + Editions + Budget & Finance, dossier racine Drive "AFTRSN Editions"
- **Aval :** Google Calendar (calendrier `hello@aftersunpeople.com`), `LUMINA-MEMORY-WRITE/WEBHOOK` (journalisation)

## Credentials

Notion account · Google Drive account · Google Calendar Hello AFTRSN · Notion API (predefined credential type, utilisé par le node HTTP Request "Notion - Injecter checklist")

## Travaux du 03.07.2026

**Corrigé et vérifié end-to-end (test EP5 — Zürich, 08.08.2026) :**
1. Upsert Venue dédupliqué — 4 bugs trouvés et corrigés : mauvais câblage vers "Notion - Chercher Venue", filtre Notion "Equals" cassé (remplacé par "Contains"), connexion résiduelle qui créait un Venue en double à chaque run, champs `venue_id` restés en mode "Fixed" au lieu de "Expression".
2. "Notion - Injecter checklist" — erreur `400 invalid_request_url` : la méthode HTTP était POST au lieu de **PATCH** (endpoint Notion `PATCH /v1/blocks/{id}/children`). Corrigé, testé, checklist injectée avec succès.

**Non résolu, en pause :**
3. "Create an event" — erreur `Invalid expression` / `No path back to referenced node` : le champ Description référence `$('Create folder').item.json.id` (lien du dossier Drive), mais la branche Dossiers Média est un cul-de-sac (voir ⚠️ ci-dessus), donc aucun chemin de connexion n'existe entre les deux nœuds. Un nœud `Merge2` a été ajouté en tentative de correction (combiner `Merge1` + `Create folder`) mais **n'est pas encore câblé** — travail interrompu pour prioriser la clarification du process métier avec Karter (cf. `W1_Decoupage_Sous-Workflows_20260703.md` et `Hermes_Clarification_20260703.md`). Le process métier clarifié depuis suggère une meilleure solution : faire pointer le calendrier vers la fiche Notion (le hub central) plutôt que directement vers le dossier Drive — supprime la dépendance croisée à la racine du problème.

**Nettoyage restant (non fait, à faire) :**
- `Merge2` orphelin à câbler ou supprimer selon la décision retenue pour `Create an event`.
- Duplicata de test : une page Venue "Mini Café Bar" en double, plusieurs pages Edition "EP5" de test créées pendant le debug — à supprimer manuellement dans Notion.

## ⚠️ Ne pas toucher sans précaution

- La branche Venue (upsert) est maintenant fiable — ne pas revenir au filtre Notion "Equals", il est cassé pour les propriétés de type titre.
- Ne pas reconnecter `Merge - Dossiers média` directement à une autre branche avec du contenu multi-item sans passer par un Merge en mode **Combine/Position** — une connexion directe multiplierait les items et recréerait le bug de duplication corrigé aujourd'hui.
- `Merge1` et le futur point d'insertion de `Merge2` : vérifier qu'aucune des deux branches ne peut produire plus d'un item à la fois avant d'utiliser le mode "Append".

## Prochaine étape suggérée

Reprendre la décision sur "Create an event" à la lumière du process métier maintenant clarifié (le calendrier doit pointer vers la fiche Notion, pas vers le dossier Drive), puis nettoyer `Merge2` et les doublons de test avant publication.
