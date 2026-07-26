---
type: raw
title: "Fiche_AFTRSN-Artist-Coordinator_20260705"
source_url: "drive:1DTtWYzGLnS4NgZfsaqlTm9OOMtKYstV4"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche workflow — AFTRSN-Artist-Coordinator

**Format :** conforme à la Bible 360° LUMINA OS §E.

---

**ID :** `VpTCflWHXkU49LDB` — **Criticité :** 🔴 (chemin critique de la deadline du 18.07 : 2 artistes confirmés)
**Statut :** 🟢 **publié le 05.07.2026** (« v1.0 — Artist-Coordinator », v1.0.1 déclenchement par statut « To contact », v1.0.2 espacement relance 2 : 4 jours après la relance 1, **v1.1 brief artistes J-7** publiée le 05.07.2026 au soir)
**Emplacement :** `Personal / 02-AFTRSN / AFTRSN-02-BUSINESS AUTOMATION / AFTRSN-PROCESS-04-Artist-Coordinator`
**Tags :** 🌞 AFTRSN · 🔄 automation · 🟢 PROD
**Déclencheur :** Schedule — jours ouvrés à 9h, fuseau **Europe/Zurich** (posé dès la création)

## Rôle

Processus transverse « Coordination artistes » : même pattern que la Coordination lieux (voir `POS-GENERIQUE_Outreach-Relances-Base-Drafts_20260704.md`), appliqué à la base **DJs / Artists** de 🎭 The Backstage. Chaque matin ouvré : artistes en statut `Prospect` avec email → **premier contact de booking**, **relance J+3**, ou **seconde relance J+7** ; rédaction par AFTRSN-Secretary (allemand + version anglaise), **brouillon** dans le Gmail **Partners** (relances dans le même fil), dates + fil écrits dans Notion, notification Telegram (🎧, zéro code technique), épisode mémoire. **Aucun envoi automatique.**

## Différences avec la Coordination lieux

- **Base pilote :** DJs / Artists (`f27a9819-d49f-489d-bf25-277e3381d564`) — champs `Email`, `1st Email`, `Follow-up 1`, `Follow-up 2`, `Gmail Thread` ajoutés le 05.07.2026 pour aligner le schéma sur Venues.
- **Statuts :** `Prospect` = réserve silencieuse (rien ne part) ; **`To contact` = déclencheur posé par Karter** → premier contact + relances ; dès qu'un artiste répond, passer à `Contacted` (ou `Confirmed`) → stoppe les relances. `Played` = historique. Le rythme d'outreach est une décision humaine, hors process (règle Karter, 05.07.2026).
- **Prompt :** email de booking avec le contexte de l'édition — 08.08.2026 au Mini Café Bar (Zürich), direction musicale Afro House / Afro Tech / Melodic Afro House, genres de l'artiste injectés ; demande de disponibilité, conditions (cachet, rider, technique) ; **jamais de montant proposé, jamais de confirmation de line-up**.
- ~~Prévu en v1.1 : brief artiste à J-7~~ → **construit, testé et publié le 05.07.2026 (v1.1)**, voir section suivante.

## v1.1 — Brief artistes J-7 (chaîne parallèle, 12 nodes ajoutés → 29 au total)

Chaîne **additive** branchée sur le même déclencheur (jours ouvrés 9h), sans toucher à la chaîne outreach v1 : `Notion - Editions (brief)` (getAll Editions) → `Editions a J-7` (Code : Status `Confirmed`/`Live`, date entre J-0 et J-7, `Lineup` non vide) → `Notion - Artistes (brief)` (getAll, executeOnce) → `Briefs du jour` (Code : par édition × artiste du lineup en `Confirmed` avec email ; **idempotence : champ `Brief J-7` (date, ajouté le 05.07.2026)** — brief déjà daté dans la fenêtre J-7 de l'édition = ignoré, un vieux brief d'une édition antérieure ne bloque pas) → `Secretary - Brief` (Execute Workflow, **mode « each »**) → `Decouper brief` → `Gmail - Brouillon brief` (**nouveau fil**, compte Partners — pas de threadId : les artistes 08.08 sont bookés par WhatsApp, souvent sans fil email) → `Notion - Noter brief` (`Brief J-7` = aujourd'hui) → `Telegram - Notifier brief` (🎤) → `Compter briefs` → `Memoire - Brief Episode` → `Resume brief`.

**Contenu du brief (prompt Secretary) :** confirmation enthousiaste, rappel date + ville, demande/confirmation : créneau de set, rider technique, logistique (arrivée, soundcheck, transport), contact jour J ; détails définitifs annoncés pour plus tard ; **si le lieu n'est pas fourni, rester générique — ne jamais inventer de nom de lieu** (la résolution du nom du lieu via la relation Venue = candidat v1.2) ; jamais de montant ni d'engagement. Allemand (Sie) + version courte anglaise, format `SUJET:`/`RESUME:`/`---DRAFT---`.

**Tests du 05.07.2026 :** brief généré pour l'édition test à J+5 (brouillon Gmail ✅, `Brief J-7` daté ✅, Telegram ✅, épisode ✅), re-run → 0 brief (idempotence ✅), chaîne v1 restée muette (artiste `Confirmed` ≠ `To contact` ✅). Données de test : fiche artiste `ZZTEST-Brief-Artist` + édition `ZZTEST-Brief-Edition` (neutralisée `Cancelled`) → corbeille par Karter, + 1 brouillon Gmail ZZTEST à supprimer.

**⚠️ v1.1 :** pour que le brief parte, l'artiste doit être lié à l'édition via **`Lineup`** (fiche Edition), en statut **`Confirmed`** avec un **email** — les artistes bookés par WhatsApp sans email ne reçoivent rien (limite connue, canal WhatsApp différé). Pour le 08.08 : consigner les artistes confirmés avec leur email dès que possible, le brief partira automatiquement autour du 01.08.

## Chaîne (17 nodes)

Identique au Venue-Coordinator : `Jours ouvres a 9h` → `Notion - Artistes en base` → `Actions du jour` → `Secretary - Rediger` → `Decouper la reponse` → `Aiguillage` (3 branches Gmail + Notion) → `Rejoindre` → `Telegram - Notifier` → `Compter` → `Memoire - Save Episode` (source_ref `AFTRSN-Artist-Coordinator`) → `Resume`.

## Construit / testé / publié

Construit le 05.07.2026 par l'API REST interne (pattern répliqué en ~1h grâce à la POS). 4 tests validés : premier contact, relance J+3, relance J+7 (même fil), à-vide (arrêt propre). Fiche `ZZTEST-Artist-Coordinator` neutralisée (corbeille par Karter) + 3 brouillons de test à supprimer par Karter.

## ⚠️ Ne pas toucher sans précaution

Mêmes précautions que le Venue-Coordinator (fiche du 04.07) : relances calculées sur la date du brouillon ; arrêt des relances = changement de Status manuel ; ne pas vider `Gmail Thread` ; garder les balises `SUJET:`/`RESUME:`/`---DRAFT---` ; sortie Notion getAll = `property_*` snake_case. **La base est vide au lancement** : le processus ne fait rien tant que Karter n'a pas créé ses cibles de booking (avec email) **en statut `To contact`** (`Prospect` = réserve, rien ne part).
