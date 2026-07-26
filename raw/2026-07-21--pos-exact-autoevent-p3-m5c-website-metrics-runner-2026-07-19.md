---
type: raw
title: "POS-EXACT_AutoEvent-P3_M5c-website-metrics-runner_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m5c-website-metrics-runner_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent P3 · M5c — Website Metrics Runner (`AFTRSN-WEBSITE-METRICS`)

**Date :** 19.07.2026 · **Marque :** AFTRSN · **Type :** POS-EXACT · **Statut :** ✅ CONSTRUIT, VALIDÉ (test contrôlé EP05 sur données réelles), **ACTIF (quotidien 07:20)**.

---

## 0. Rôle
Deuxième runner de la couche **📊 Metrics** (chantier Contents & Metrics). Il remonte automatiquement, pour chaque édition dont l'événement Wix est publié, deux chiffres dans la base **Website Metrics** : les **visites de la page événement** et le nombre d'**inscriptions RSVP « oui »**, reliés à la ligne Website Content et à l'édition. Une fois l'événement en ligne, les chiffres arrivent seuls.

## 1. Runner
- **ID :** `3ltdhOLhIdfHJ2At` · **15 nœuds** · **quotidien 07:20** (cron `20 7 * * *`, TZ Europe/Zurich) · Error-WF Sentinel `xzaH0uWy0idKVphF` · cred Notion `sCsj206WVkNxO0Jl` + cred Wix `2SjAjkAPXNNjUQYL` (clé API = **All permissions** → Analytics read + Events guest-list read couverts).
- **Chaîne :** `Daily 07:20` → `Build editions query` (Code : filtre `Wix event ID` non vide ET `Status ≠ Cancelled`) → `List editions` (HTTP query Editions `dad7f826…`) → `Explode editions` (Code, multi-items : editionId, editionTitle, wixEventId, eventType, contentId = 1ᵉʳ élément de la relation `Website`) → **boucle `Loop editions`** (SplitInBatches 1) : `Get Wix event` (GET `/events/v3/events/{id}`) → `Prepare` (Code : garde d'état + construit les bodies) → `Active?` (IF) → `Get visits` (POST Semantic Model `query-data`) → `Get RSVP` (POST `/events/v2/rsvps/count`) → `Build upsert` (Code : props JSON bruts + findBody par TITRE) → `Find metrics row` (HTTP query Website Metrics `4f2a9029…`) → `Metrics exists?` (IF) → `Update metrics row` (PATCH `/v1/pages/{id}`) / `Create metrics row` (POST `/v1/pages`) → retour boucle. La branche `Active? = false` retourne aussi à la boucle (garde-fou de drainage, pas de blocage).

## 2. Règles fonctionnelles
1. **Source = les éditions, pas le contenu** : le `Wix event ID` (qui débloque la relève) vit sur l'**édition** (écrit par M4). Le runner itère les éditions qui en portent un et dont le `Status ≠ Cancelled` (Done accepté).
2. **Garde « événement actif »** (nœud `Prepare`) : on relève seulement si l'objet Wix a un `publishedDate` (donc **publié, pas draft**) ET `status ≠ CANCELED`. Un événement **ENDED est accepté** (le trafic « recap » continue). Sinon → skip, retour boucle.
3. **Visites = page événement SEULE, mesure sessions, cumulé depuis mise en ligne** (décisions Karter). L'API publique **Analytics Data** ne fait que le site entier (pas de filtre page) → **écartée**. On passe par l'**API Semantic Model**, modèle **`traffic`** (`cad7fd34-2c8b-4dda-8296-3f9d47fb484d`), filtre **par chemin de page** `traffic.page_url_from = /event-details/<slug>` (le `<slug>` vient de l'événement Wix). Fenêtre = de la mise en ligne à aujourd'hui, **plafonnée à 62 jours** (rétention analytics Wix).
4. **RSVP = comptage « YES »** : `POST /events/v2/rsvps/count` body `{filter:{eventId, status:'YES'}}`. Un événement **Paid/EXTERNAL** (billetterie externe) renvoie **0** (pas de RSVP côté Wix) — normal.
5. **Upsert par TITRE** `<edition> · Event page` : la ligne Metrics porte le même titre que la ligne Website Content ; create au 1ᵉʳ passage, update ensuite (1 ligne par édition, jamais de doublon — prouvé).
6. Champs écrits : `Visits` (number), `RSVP` (number), `As-of` (date du jour), relations `Content` (→ ligne Website Content via la relation `Website` de l'édition) + `Edition`.
7. **Écritures Notion en HTTP brut** (POST/PATCH `/v1/pages`), corps JSON construits en nœud Code (`JSON.stringify`) — règle canonique (number/date jamais via le nœud Notion).

## 3. Endpoints Wix (verrouillés avant build, prouvés live)
- **Visites** : `POST https://www.wixapis.com/analytics/semantic-model/v3/semantic-models/query-data`, body `{ semanticModelId, interval:{start,end} (ISO datetime, REQUIS), fields:['traffic.sessions_count'], filters:[{field:'traffic.page_url_from', prefix:'IS', condition:'EQUAL', values:['/event-details/'+slug]}], totalsIncluded:true }`. Lecture = `totals.fields['traffic.sessions_count'].numericValue`. Modèle découvert via `GET …/semantic-models` puis inspecté via `GET …/semantic-models/{id}`.
- **RSVP** : `POST https://www.wixapis.com/events/v2/rsvps/count` → `{count}`.
- **État événement** : `GET https://www.wixapis.com/events/v3/events/{id}` → `slug` (→ chemin `/event-details/<slug>`), `status` (UPCOMING/STARTED/ENDED/CANCELED), `publishedDate` (absent = draft), `registration.type`.
- Auth = credential Custom Auth (injecte `Authorization` + `wix-site-id`), pattern nœud HTTP : `authentication:genericCredentialType`, `genericAuthType:httpCustomAuth`.

## 4. Preuves live (19.07)
- Croisière : `List editions` → 0 édition avec `Wix event ID` actuellement → 0 action (exéc. `21418`).
- **Test contrôlé** : `Wix event ID` d'EP05 pointé temporairement sur l'événement réel **after-sun-2** (`b87cadf1-b367-4a5c-a5d5-e5300f6c6079`, publié, ENDED, RSVP) →
  - **create** ligne `EP05 - AFTRSN - HOME · Event page` = **Visits 532 · RSVP 32 · As-of 19.07**, relations Content+Edition (exéc. `21437`) ;
  - **update** de la même ligne au run suivant, pas de doublon (exéc. `21442`).
- Endpoints prouvés en direct avant build : `/event-details/after-sun-2` filtre EQUAL = 529 sessions (572 vues / 441 uniques) ; RSVP YES = 32.
- **État remis propre** : EP05 `Wix event ID` revidé, webhook de test retiré. Ligne démo `EP05 · Event page` conservée (choix Karter) — données after-sun-2, pas le vrai événement d'EP05.

## 5. Pièges / leçons de ce build
- **Visites par page ≠ Analytics Data API** : cette API est site-entier, sans filtre page. Le **modèle sémantique `traffic`** expose la dimension `traffic.page_url_from` (Page Path) qui, elle, filtre par page.
- **Chemin d'une page événement Wix = `/event-details/<slug>`** (sous-pages `…/form`, `…/thank-you-messages/yes`). Filtrer en `EQUAL` = page principale seule ; `START_WITH` inclurait les sous-pages.
- **Garde d'état par `publishedDate` + `status`** : `publishedDate` présent = publié (pas draft) ; plus robuste qu'un flag « draft ».
- **Rétention analytics 62 jours** : au-delà, plafonner le `start` (les données plus anciennes n'existent plus).
- **Trigger planifié non déclenchable par API** : pour tester, réactiver + basculer le schedule en 1 min, ou ajouter un webhook temporaire (retiré ensuite).
- **Tester l'idempotence sur le 2ᵉ run** (le create réussit toujours ; c'est l'update qui prouve l'absence de doublon).

## 6. Reste (couche Metrics)
- **Instagram Metrics** : saisie manuelle à date figée (décision Karter, pas d'API Meta) — rien à construire.
- Suite moteur : **M6 Instagram**.
