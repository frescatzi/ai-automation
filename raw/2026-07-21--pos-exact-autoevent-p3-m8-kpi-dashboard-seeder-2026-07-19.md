---
type: raw
title: "POS-EXACT_AutoEvent-P3_M8-kpi-dashboard-seeder_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m8-kpi-dashboard-seeder_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · M8 (moitié 1) — KPI Roller + Marketing & KPI + Dashboard hébergé + Metrics Seeder

Date : 19.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Moteur : n8n LUMINA OS (self-hosted Hetzner/Coolify, `https://n8n.aftersunpeople.com`) × Notion « The Backstage »
Statut : ✅ CONSTRUIT, TESTÉ LIVE, VALIDÉ (Karter), ACTIF. Moitié 1 de M8 (remontée KPI + dashboard + seeding metrics). Reste moitié 2 = récap de convergence.

## 1. Objectif
Donner à Karter un pilotage marketing par édition : une base **💫 Marketing & KPI** (1 ligne/édition), un **dashboard vivant dans Notion**, et des bases Metrics **pré-remplies** pour la saisie. Aucun automate n'écrit Approved/Done ; cibles/status = Karter.

## 2. Ce qui a été construit

### 2.1 Base 💫 Marketing & KPI (restructurée)
- DB `63bb41f433d1452e98cc51cf3253682b` · ds `f42933a8-8099-4e6e-b943-ccd4765c1fc3`.
- **Modèle = 1 ligne par édition** (choix Karter après avoir vu qu'1-ligne-par-métrique = 3 pages/édition + scroll infini + titres redondants).
- Titre `Initiative` = « EPxx · DD.MM.YYYY ». Clé d'upsert = **`Edition key`** (text, ex. « EP05 »), jamais le titre ni la relation.
- Colonnes : `Newsletter open rate`, `Registrations` (ex-« RSVP », renommé), `Website visits`, `Instagram reach`, `Target NL/Reg/Web/IG`, `NL/Reg/Web/IG vs prev`, `Edition` (relation), `Edition date`, `Status`.
- Vues : « Éditions (récent en haut) » (tri `Edition date` DESC) + vue liée filtrée par `Edition key` sur la page d'une édition. Moyenne = pied de colonne natif Notion.

### 2.2 KPI Roller — `AFTRSN-KPI-ROLLER` `SCtfpndW8KULi616` (11 nœuds, quotidien 07:25)
- Schedule 07:25 → List newsletter/website/instagram/editions (HTTP Notion) → Build KPI lines (Code) → Find (par `Edition key`) → Prep write (zip) → IF exists → Update / Create.
- **1 ligne par édition**, calcule open rate = Opened/Delivered, Registrations = RSVP, Visits, Reach, + **Δ vs édition précédente** (éditions triées par date). Update = dérivés + Δ seuls (jamais Target/Status/Objective/titre). Relation Edition résolue par titre.
- **Durci** : ignore les lignes Metrics **vides** (le seeder pose des placeholders) — ne compte une base que si valeur non nulle, sinon des « 0 » fictifs pollueraient le dashboard.

### 2.3 Dashboard — `AFTRSN-KPI-DASHBOARD` `BlisYeB3rbEX4LVa` (webhook, actif)
- Webhook GET `path=kpi-dashboard` → Load KPI (HTTP Notion) → Build HTML (Code) → Respond to Webhook (text/html). URL publique `https://n8n.aftersunpeople.com/webhook/kpi-dashboard`, lit Marketing & KPI **en direct** à chaque appel.
- Intégré en **bloc `<embed>`** sur la page Notion « Marketing Dashboard » (`3a224f2f-ef0c-81e5-ae93-c19ad9b8c374`, sous Metrics).
- **Page hybride (progressive enhancement)** : rendu **statique côté serveur** (cartes valeurs/cibles/progression, 4 mini-courbes SVG avec valeurs, table + moyenne) qui s'affiche dans le sandbox Notion ; **+ script** qui, si le JS s'exécute (onglet navigateur), rebâtit la version **interactive** (cartes cliquables → grande courbe, survol tooltips, toggle « Compare with other events »). Bascule statique→interactif APRÈS rendu réussi + try/catch (jamais de page blanche). Libellés anglais.

### 2.4 Metrics Seeder — `AFTRSN-METRICS-SEEDER` `ZufKDJqlMPHEfxGD` (7 nœuds, scan 15 min)
- Schedule 15 min → List editions (Status ≠ Cancelled) → Explode (3 items/édition) → Find row (HTTP, DB dynamique) → Prep → IF exists → Create (branche false).
- Garantit **une ligne placeholder titrée** dans les 3 bases Metrics : `<édition> · Instagram` / `· Newsletter` / `· Event page`, Edition reliée, chiffres vides. Idempotent (create-if-missing). But = saisie manuelle guidée (surtout **Instagram Metrics**, sans runner auto). Titre `· Event page` aligné sur M5c → upsert commun.

## 3. Difficultés
- **Notion embed = JavaScript désactivé** (iframe sandboxé) → dashboard interactif impossible DANS Notion (première tentative = page blanche puis shell vide).
- Modèle 1-ligne-par-métrique = 3 pages/édition + redondance de titres + scroll infini à 20-30 éditions.
- Graphiques natifs Notion = fonctionnalité payante (Business/IA) + rendu pauvre.
- Placeholders vides du seeder → risque de « 0 » fictifs au dashboard.
- Notion MCP ne sait pas archiver les vieilles lignes ; limite horaire de requêtes SQL `query_data_sources` (plan gratuit).

## 4. Solutions
- **Dashboard hébergé par webhook n8n** (infra publique Hetzner/Coolify) + **page hybride** : statique dans Notion, interactif au navigateur, une seule URL. Durci (bascule après rendu + try/catch).
- **Restructuration 1-ligne-par-édition**, colonnes par KPI, moyenne native, tri récent-en-haut, titre court épisode+date, clé stable `Edition key`.
- **KPI Roller durci** pour ignorer les lignes vides.
- Purge des vieilles lignes via mini-workflow jetable `PATCH archived:true` (créé, exécuté, supprimé). Vérifications via API HTTP / exécutions n8n plutôt que SQL.

## 5. Lessons
- **Notion `<embed>` n'exécute pas de JS** → pour de l'interactif « dans Notion », il faut soit du natif (payant), soit une page hybride servie en externe (statique dans l'iframe, interactive au navigateur).
- **Héberger un dashboard vivant = un webhook n8n qui génère le HTML depuis la source** ; embed l'URL dans Notion ; mise à jour à chaque chargement, sans snapshot ni abonnement.
- **Un agrégateur doit ignorer les placeholders vides** (compter seulement les valeurs non nulles) sinon les lignes de seeding faussent les métriques.
- **Une clé d'upsert stable et dédiée** (`Edition key`) vaut mieux qu'un titre qui embarque une date.
- **Voir le rendu réel tôt** : le client sait immédiatement dire « promis pas délivré » — montrer dans l'outil cible (Notion), pas juste une maquette.

## 6. Preuves live
- KPI Roller : EP05 → open rate 78.6 / Registrations 32 / Website visits 532 ; Target 80/100/600 préservés ; idempotent (Update, 0 doublon).
- Dashboard : URL 200, statique rendu dans Notion (vérifié capture Karter), interactif confirmé au navigateur (cartes + courbe + toggle).
- Seeder : 5 éditions × 3 bases = 15 items → 10 créées puis 0 (idempotent) ; Instagram Metrics pré-rempli (EP02/03/05/06/07).

## 7. Validation
✅ Karter 19-07 (dashboard « c'est parfait », toggle renommé, 3 points d'amélioration livrés).

## 8. Reste
M8 moitié 2 = récap de convergence (canaux d'une vague tous Approved → récap « prêt à diffuser », zéro envoi auto ; cadrage : où atterrit le récap). Puis M9 clôture + test E2E EP05.
