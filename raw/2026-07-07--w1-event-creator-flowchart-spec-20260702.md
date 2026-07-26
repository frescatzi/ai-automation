---
type: raw
title: "W1_Event-Creator_Flowchart_Spec_20260702"
source_url: "drive:1nquLClc8zysH51HM24_wnZJ4gp9d6gLX"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# W-1 · AFTRSN-Event-Creator — Spec node par node
**Statut :** Draft v2 — réconciliée avec le Notion réel ("The Backstage") le 02.07.2026
**Base :** PASSATION_AFTRSN_AutomationArchitecture_20260702.md — ⚠ l'ADDENDUM 2 de ce document (DB Events/Venues/Artists/Event Tasks) était une proposition théorique. Le Notion réel existe déjà sous un autre nom et un autre schéma (voir §0). Cette version écrit dans les DB **existantes**, n'en crée aucune nouvelle.
**Décision Karter (02.07.2026) :** n8n devient la couche d'automatisation immédiate de ce système — pas un projet parallèle.

---

## 0 · Ce qui existe déjà dans Notion ("The Backstage")
Confirmé par lecture directe le 02.07.2026 — ne pas re-proposer de nouvelles DB, écrire dans celles-ci :

| DB Notion réelle | Rôle | Champs clés |
|---|---|---|
| **Editions** (pivot central, pas "Events") | 1 page = 1 édition d'AFTER SUN | `Edition` (title), `City` (text), `Date`, `Status` (`Planning`·`Confirmed`·`Live`·`Done`·`Cancelled`), `Capacity target`, `Venue` (relation), `Lineup` (relation → DJs/Artists), `Lineup notes`, `Recap link`, + relations vers les 6 DB ci-dessous |
| **Venues** | Annuaire lieux | `Venue` (title), `Status` (`Prospect`·`In discussion`·`Confirmed`·`Active`), `Capacity`, `City` (relation), `Contact`, `Lead` (multi-select : Katel/Jack/Jacques/Sara), `First Contact`/`1st Email`/`Follow-up 1`/`Follow-up 2` (dates), `Notes` |
| **DJs / Artists** | Annuaire artistes | `DJ / Artist` (title), `Status` (`Prospect`·`Contacted`·`Confirmed`·`Played`), `Genres` (multi-select), `Home city` (relation), `Contact`, `Notes` |
| **Budget & Finance** | Lignes budgétaires | relation → Editions |
| **Content & Social** | Contenu par édition | relation → Editions |
| **Marketing & KPI** | Métriques par édition | relation → Editions |
| **Partners & Sponsors** | Sponsors/partenaires | relation → Editions |
| **Media Assets** | Photos/vidéos | relation → Editions |
| **Cities / Expansion** | Villes cibles | relation → Venues, DJs/Artists |

Il n'existe **pas** de DB "Event Tasks" séparée — la checklist opérationnelle vit directement dans le corps de la page Edition (cf. page "📋 EDITION TEMPLATE — duplicate to start"), organisée en 7 sections + post-event.

**Convention de nommage média (canon, Brand Source of Truth) :** dossiers `YYYYMMDD_AFTSN_Photos` et `YYYYMMDD_AFTSN_Videos`, chacun avec sous-dossiers `Raw` et `Edited`. À respecter strictement dans Drive.

## Objectif
Une commande manuelle (ou déclenchée par confirmation venue) crée : la page Edition dans Notion avec sa checklist, l'arborescence Drive, l'entrée Calendar, et amorce les lignes Budget & Finance — en un seul run, sans dupliquer une structure qui existe déjà.

## Entrée (trigger)
`Execute Workflow Trigger`, appelé par `AFTRSNMaestro`.

```json
{
  "edition_name": "AFTER SUN — Zürich 08.08.2026",
  "city": "Zürich",
  "event_date": "2026-08-08",
  "venue_name": "Le Rex",
  "capacity_target": 400,
  "requested_by": "Karter"
}
```
⚠ Piège `jsonExample` : garder cet exemple à jour à chaque champ ajouté.

## Nodes

| # | Node | Type n8n | Description |
|---|---|---|---|
| 1 | Trigger Maestro | Execute Workflow Trigger | Reçoit la commande |
| 2 | Valider Input | IF | `edition_name`, `city`, `event_date` présents. Sinon → branche erreur (13) |
| 3 | Normaliser & slug | Set / Code | `folder_name`, dates dérivées (`campaign_start = event_date - 21j`, `artist_brief = event_date - 7j`), noms de dossiers média au format canon |
| 4 | Drive — Créer dossier édition | Google Drive (Create Folder) | Parent = dossier racine "AFTRSN Editions" |
| 5 | Drive — Créer sous-dossiers média | Google Drive (Create Folder × N) | `{date}_AFTSN_Photos/Raw`, `{date}_AFTSN_Photos/Edited`, `{date}_AFTSN_Videos/Raw`, `{date}_AFTSN_Videos/Edited` — format canon exact |
| 6 | Calendar — Créer event | Google Calendar (Create Event) | Titre = `edition_name`, date = `event_date`, description = lien dossier Drive |
| 7 | Notion — Upsert Venue | Notion (Search dans DB Venues + Create si absent) | Si `venue_name` fourni : cherche par titre, crée en `Status=Prospect` si absent, `City` en relation |
| 8 | Notion — Créer Edition | Notion (Create Page, DB **Editions**) | `Edition`=nom, `City`, `Date`, `Status=Planning`, `Capacity target`, `Venue` (relation vers 7 si trouvée) |
| 9 | Notion — Injecter checklist | Notion (Append blocks à la page 8) | Copie la structure de "EDITION TEMPLATE" (7 sections + post-event) dans le corps de la page — pas de DB séparée |
| 10 | Notion — Amorcer Budget & Finance | Notion (Create Pages, DB Budget & Finance, relation → 8) | Lignes vides pré-nommées : Venue, Artists, Production, Marketing, Media (correspond à la section 3 du template) |
| 11 | Mémoire — Save Episode | HTTP Request → LUMINAMEMORYWEBHOOK | `collection=episodic`, payload = résumé + URLs (indépendant de Notion, alimente pgvector) |
| 12 | Agréger résumé | Set | `{status, notion_url, drive_url, calendar_url}` |
| 13 | Erreur → réponse | Set + Respond to Webhook | Si (2) échoue ou 4–10 échouent : `{status:"error", step, message}`, pas d'écriture partielle silencieuse |
| 14 | Répondre à Maestro | Respond to Webhook | Renvoie le résumé (12) |

## Ce qui change vs. la v1 (ADDENDUM 2 théorique)
- Pas de DB "Events" à créer → écrire dans **Editions**.
- Pas de DB "Event Tasks" → checklist en contenu de page, comme le fait déjà le template existant.
- Statuts alignés sur les valeurs réelles (`Planning` pas `Idea`, `Prospect`/`In discussion`/`Confirmed`/`Active` pour Venues).
- Budget : les lignes vivent dans **Budget & Finance** (relation), le Sheets détaillé devient optionnel/secondaire plutôt que la source de vérité.
- Nommage dossiers média strictement conforme au canon (`YYYYMMDD_AFTSN_Photos/Videos`, `Raw`/`Edited`).

## Gestion d'erreur
Inchangée : pas d'écriture partielle silencieuse, aucune action destructive, HTTP Request pour tout appel Claude (jamais le node Anthropic natif — 404 connu).

## Pré-requis avant implémentation
- Credential Google OAuth (Drive + Calendar) créé et testé dans n8n
- Connecteur n8n autorisé côté Claude (en cours — statut à revérifier)
- Dossier racine "AFTRSN Editions" existant sur Drive
- Accès Notion confirmé (déjà opérationnel — testé le 02.07.2026)

## Prochaine étape
Construire ce workflow dans n8n dès que le connecteur est autorisé, en pointant vers les vraies data-source URLs Notion listées en §0.
