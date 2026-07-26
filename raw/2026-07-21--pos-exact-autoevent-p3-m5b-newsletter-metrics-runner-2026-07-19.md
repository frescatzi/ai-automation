---
type: raw
title: "POS-EXACT_AutoEvent-P3_M5b-newsletter-metrics-runner_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m5b-newsletter-metrics-runner_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent P3 · M5b — Newsletter Metrics Runner (`AFTRSN-NEWSLETTER-METRICS`)

**Date :** 19.07.2026 · **Marque :** AFTRSN · **Type :** POS-EXACT · **Statut :** ✅ CONSTRUIT, TESTÉ LIVE (données réelles EP05), VALIDÉ (Karter 19.07, exécution finale à sa demande), **ACTIF (quotidien 07:15)**.

---

## 0. Rôle
Premier runner de la couche **📊 Metrics** (chantier Contents & Metrics) : il remonte automatiquement les statistiques des campagnes newsletter Wix dans la base **Newsletter Metrics**, reliées à leur ligne Content et à l'édition. L'humain envoie la campagne dans Wix et marque la ligne Content `Sent` ; les chiffres arrivent seuls.

## 1. Runner
- **ID :** `4Jpo2RzBE2K6i2mE` · **14 nœuds** · **quotidien 07:15** · Error-WF Sentinel `xzaH0uWy0idKVphF` · cred Notion `sCsj206WVkNxO0Jl` + cred Wix `2SjAjkAPXNNjUQYL` (perm stats vérifiée : `GET /email-marketing/v1/campaigns` → 200).
- **Chaîne :** `Daily 07:15` → `Build sent query` (Code : rows `Status=Sent` ET `Sent date ≥ J-8`) → `List sent newsletters` (HTTP query Newsletter Content `190dd201…`) → `Explode rows` (Code, multi-items : contentId, title, edition, editionCode = 1ᵉʳ token, campaignId existant, sentDate, editionRelId) → **boucle `Loop rows`** (SplitInBatches 1) : `Get campaigns` (GET `/email-marketing/v1/campaigns`) → `Match campaign` (Code) → `Get campaign stats` (GET `/campaigns/statistics?campaignIds={id}`) → `Build metrics upsert` (Code : props JSON bruts + findBody par TITRE) → `Find metrics row` (HTTP query Newsletter Metrics `8c2a83a4…`) → `Metrics row exists?` (IF) → `Update metrics row` (PATCH `/v1/pages/{id}`) / `Create metrics row` (POST `/v1/pages`) → `Write Campaign ID to content` (PATCH sur la ligne Content) → retour boucle.

## 2. Règles fonctionnelles
1. **Fenêtre de relève : quotidienne, de `Sent date` à J+7** (décision Karter). Drainage **par date** (pas de mutation) : passé J+7, la ligne sort du filtre toute seule.
2. **Matching campagne↔ligne** : `Campaign ID` prérempli sur la ligne Content = prioritaire ; sinon **le titre de la campagne Wix doit commencer par le code édition** (ex. « EP05… », convention Karter) ; si plusieurs candidates → la plus proche de `Sent date` ; aucune → skip (retry le lendemain). L'ID matché est réécrit sur la ligne Content.
3. **Upsert par TITRE** : la ligne Metrics porte le même titre que la ligne Content ; create au 1ᵉʳ passage, update ensuite (1 ligne par campagne, jamais de doublon — prouvé).
4. Champs écrits : `Delivered / Opened / Clicked / Bounced` (uniques, source `statistics[0].email`), `As-of` = jour de relève, relations `Content` + `Edition`.
5. **Écritures Notion en HTTP brut** (POST `/v1/pages`, PATCH), corps JSON construits en nœud Code (`JSON.stringify`) — règle canonique (jamais d'objet littéral dans une expression n8n ; nœud Notion non utilisé pour number/date).
6. Lien analytics (format confirmé par Karter) : `https://manage.wix.com/dashboard/{siteId}/email-marketing/analytics/{campaignId}`.

## 3. Preuves live (19.07)
- Croisière : 0 ligne Sent → 0 action (exéc. `20886`).
- Chemin complet campagne brouillon : **create** ligne Metrics (exéc. `20900`) puis **update** au tick suivant, pas de doublon (exéc. `20914`).
- **Données réelles** : campagne `46c77583-99d2-468f-a218-81c518e3c136` (la vraie newsletter du **26.06 → EP05** ; attribution corrigée par Karter — pas EP06) → **Delivered 14 · Opened 11 · Clicked 0 · Bounced 0** (+ landingPage opened 7) écrits dans Newsletter Metrics avec relations.
- **État final en production** : ligne Content `EP05 … · Announce · Newsletter` = Sent (19.07) + Campaign ID/link ; ligne Metrics live, rafraîchie chaque matin jusqu'au 26.07 puis arrêt automatique. EP06 remis à zéro après le test mal attribué.

## 4. Pièges / leçons de ce build
- **Trigger planifié non déclenchable par API** : pour tester ou exécuter à la demande, basculer temporairement le schedule en 1 min (1 tick) puis remettre le quotidien.
- Les stats d'une campagne **non envoyée** reviennent sans bloc `email` → défauts à 0 (ne pas planter).
- Vérifier une permission API par un **GET de lecture jetable** avant de construire le runner qui en dépend.

## 5. Reste (couche Metrics)
- **Website Metrics runner** (à construire) : 1×/jour, visites (`Analytics · GetAnalyticsData`) + inscriptions (`RSVP · CountRsvps`), **tant que l'événement Wix est actif (ni cancelled ni draft CÔTÉ WIX)** — chercher les endpoints exacts dans le spec avant build.
- **Instagram Metrics** : saisie manuelle à date figée (décision Karter, pas d'API Meta) — rien à construire.
