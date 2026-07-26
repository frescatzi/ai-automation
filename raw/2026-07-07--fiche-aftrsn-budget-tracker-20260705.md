---
type: raw
title: "Fiche_AFTRSN-Budget-Tracker_20260705"
source_url: "drive:1icnJT8XdGsIssCvWmfaT2TOpwdEMxdG6"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche workflow — AFTRSN-Budget-Tracker (W-6)

**Format :** conforme à la Bible 360° LUMINA OS §E.

---

**IDs :** orchestrateur `AFTRSN-Budget-Tracker` `2mEakXhOGBs3dlyX` (7 nodes) + sous-workflow `AFTRSN-W6.1 — Suivi Edition` `Qk0b1xwqkr0Pmtpk` (21 nodes) — **Criticité :** 🟠 (pilotage financier, pas de chemin critique événementiel)
**Statut :** 🟢 **publiés le 05.07.2026** (« v1.0 — Budget-Tracker » / « v1.0 — W6.1 Suivi Edition »)
**Emplacement :** `Personal / 02-AFTRSN / AFTRSN-02-BUSINESS AUTOMATION / AFTRSN-PROCESS-05-Budget-Tracker`
**Tags :** 🌞 AFTRSN · 🔄 automation · 🟢 PROD
**Déclencheur :** Schedule — **vendredi 9h**, fuseau Europe/Zurich (décision Karter 05.07.2026)

## Rôle

Suivi budget continu par édition (dernier morceau de l'Étape 2). Chaque vendredi :
1. Lit la base **Editions** 🎭 ; retient les éditions avec date, hors `Done`/`Cancelled`.
2. Par édition (sous-workflow W6.1, mode « une exécution par item ») :
   - trouve le **fichier budget Google Sheets** de l'édition dans le dossier Drive `Accounting & Finance` (convention `YYYYMMDD_Event-Budget_<Slug>`) ; **le crée depuis le template si absent** (self-healing) ;
   - lit les onglets `Event Budget` (10 catégories, prévu/réel) et `Event Revenue` (Sponsors, Program Ads, Billetterie, Bar & Produits, Autres) ;
   - calcule totaux, P&L prévu/réel, **coût et revenu par tête** (Capacity target de l'édition), **alertes ⚠️ ≥80 % / 🔴 ≥100 %** par catégorie (+ 🔴 dépense sans budget prévu) ;
   - met à jour le **résumé Notion** : 9 lignes Budget & Finance par édition (Venue, Artists, Production, Marketing, Media + Revenus - Billetterie / Bar & Produits / Sponsors & Autres + P&L), champs `Planned`/`Actual`/`Category` — création si manquantes, sinon mise à jour ;
   - fait analyser les chiffres par **AFTRSN-Comptable-Finance** (publié à cette occasion → **DA-001 réglée**) : lecture générale, points de vigilance, recommandation, ~120 mots sans code technique ;
   - notifie **Telegram** (chiffres + alertes + analyse + lien fichier) ;
   - **clôture auto à J+3** : édition passée depuis ≥3 jours → rapport de clôture + `Status = Done` sur la fiche Edition.
3. Journalise un épisode `aftrsn_memory` (`episodic`, source_ref `AFTRSN-Budget-Tracker`).

**Comportement à vide :** aucune édition à suivre → arrêt silencieux après `Editions a suivre`.

## Répartition des rôles (vision Karter, 05.07.2026)

- **Le fichier Excel/Sheets = le détail** (source de vérité du budget, travaillé à la main par Karter/core team).
- **Notion = le résumé** (pub, artistes, bar, sponsoring, P&L — lisible d'un coup d'œil sur l'édition).
- **Comptable-Finance = la surveillance** (moyennes par tête selon capacité, dérives, anticipation des coûts, arbitrages type « 3 DJs au lieu de 4 », lieu trop cher).

## Template budget

- **`TEMPLATE_Event-Budget`** (Google Sheets natif) : `1VVGaXKJWpprnMMWwKPKJMHNClZZPWylHZ4DWwELfWEA`, dossier `Departments/Accounting & Finance` (`1wwbZ5N7R_sctE8isJJWq4wCii_DBv_BJ`, compte **Hello**).
- Créé le 05.07.2026 par conversion de `20260328_Event-Budget_MiniCafe.xlsx` (intact) : montants saisis vidés, formules de sous-totaux conservées, artistes → `Main Artist 1/2/3`, commentaires spécifiques retirés.
- 4 onglets : `Event Budget`, `Event Revenue`, `Event Profit Summary` (formules), `Chart Data` (dérivé ; affiche `#DIV/0!` tant que le budget est vide — cosmétique).
- **Lignes lues par le tracker (ne pas déplacer sans mettre à jour le node `Chiffres`)** — Event Budget : catégories en lignes 5, 14, 22, 27, 32, 37, 48, 57, 65, 71 (colonnes D=prévu, E=réel) ; Media = lignes 44–46 ; Event Revenue : lignes 5, 10, 18, 23, 30 (colonnes H=prévu, J=réel).

## Dépendances & credentials

- **Notion** `Notion account` (`xsBRVxaZ6hhVJWIx`) — DB Editions `dad7f826-…`, DB Budget & Finance `05d7f280-…`
- **Google Drive/Sheets** : credential Drive **HELLO** (`P7erICUpzKjuOV9E`) réutilisée pour l'API Sheets via HTTP Request (le scope Drive couvre Sheets) — **API Sheets activée le 05.07.2026** sur le projet `909839129866`
- **AFTRSN-Comptable-Finance** (`F1wsXbsYZdEkwQ3n`), contrat `{query}` — **publié le 05.07.2026** (« v1.0 — Comptable-Finance »)
- **LuminaOsBot - Telegram** (`5kTYhDVm2UDpSDIE`), chat `776345147` — **LUMINA-MEMORY-WRITE** (`zu4jfZbmDz8trQLl`)
- Aucun credential saisi par l'agent.

## Construit / testé / publié le 05.07.2026

Construit par l'API REST interne n8n (2 POST). Tests : copie du template ✅, fichier existant réutilisé ✅, création des 9 lignes Notion ✅ puis mise à jour ✅, seuils ⚠️ 85 % / 🔴 350 % détectés ✅, rapport de clôture + `Status=Done` ✅, **run orchestrateur réel multi-éditions** (ZZTEST hebdo + **EP4 clôturée pour de vrai**, décision Karter) ✅, épisode mémoire ✅.

## Difficultés rencontrées / Solutions / Leçons

| Difficulté | Solution | Leçon |
|---|---|---|
| batchGet Sheets : 2 paramètres `ranges` déclarés dans le node HTTP → n8n n'en a envoyé qu'un (dernier gagne), le workflow lisait Event Revenue comme s'il était le budget, silencieusement | Ranges inline dans l'URL (`?ranges=…&ranges=…` encodé) | n8n sérialise les query params en objet : **jamais de paramètre de requête dupliqué** dans un node HTTP — les mettre dans l'URL |
| Mes vérifications booléennes `raw.includes('"NomNode"')` disaient tous les nodes exécutés | Décoder `resultData.runData` (flatted) et ne regarder que ses clés | Le payload d'exécution contient aussi le workflowData : chercher un nom de node dans le texte brut ne prouve rien |
| Publication Comptable-Finance : 409 « conflict with one of the webhooks » (chatTrigger copié d'un autre agent, webhookId dupliqué) | Régénérer `webhookId` (UUID) sur le chatTrigger puis activer | Un workflow dupliqué garde les webhookIds d'origine : les régénérer avant publication |
| Sous-workflow appelé refusé à l'exécution : « Workflow is not active » | Publier le callee (CF) avant de tester l'appelant | Sur cette instance, un sous-workflow doit être **publié** pour être appelé — même en test manuel |
| API Sheets non activée (403) sur le projet Google | Activation par Karter (passkey requis, l'agent ne peut pas s'authentifier) | Toute nouvelle API Google = vérifier l'activation projet **avant** de construire le test ; prévoir l'aller-retour humain |
| Édition de test en `Confirmed` = retraitée chaque vendredi | ZZTEST repassée à `Cancelled` en fin de test | Toujours neutraliser les données de test **dans le même statut que le process les ignore** |

## ⚠️ Ne pas toucher sans précaution

- Les numéros de lignes du template sont câblés dans le node `Chiffres` (W6.1). Ajouter des lignes au template = mettre à jour le mapping.
- La clôture (`Status=Done`) est déclenchée par la **date + 3 jours**, pas par une action humaine. Une édition qui doit rester ouverte plus longtemps doit voir sa date ou le code ajustés.
- Les lignes Notion sont retrouvées par **titre** (exact puis contains). Renommer les lignes P1.5 (Venue/Artists/Production/Marketing/Media) casserait le rattachement (des doublons seraient créés).
- Le champ `Status` des lignes Budget & Finance (Planned/Committed/Paid…) n'est **pas** touché par le tracker — il reste l'outil de gestion manuel de Karter.
- Le tracker ne repasse jamais sur une édition `Done`/`Cancelled` : pour re-suivre, repasser le statut à `Confirmed`.

## Nettoyages en attente (Karter)

- Fiche Notion `ZZTEST-Budget-Tracker` (neutralisée en `Cancelled`) + ses 9 lignes Budget & Finance → corbeille.
- Fichier Drive `20260720_Event-Budget_ZZTESTBudgetTracker` → corbeille.
- **DA-007** : supprimer `ZZZ_20260705_tmp_budget_test` (`BkQlAcRRRmJQp0tU`).
- EP4 : fichier `20260626_Event-Budget_EP4` créé vide — à remplir a posteriori si les chiffres réels existent (le tracker ne repassera pas dessus, EP4 est `Done`).
