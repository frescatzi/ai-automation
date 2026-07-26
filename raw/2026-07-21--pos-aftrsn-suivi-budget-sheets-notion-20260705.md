---
type: raw
title: "POS-AFTRSN_Suivi-Budget-Sheets-Notion_20260705"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_suivi-budget-sheets-notion_20260705.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS AFTRSN — Suivi budget piloté par Google Sheets + Notion, alertes de seuils, clôture auto (Budget-Tracker)

Procédure vérifiée le 05.07.2026 — construction, test (fichier absent/présent, création/mise à jour des lignes, seuils, clôture, run multi-éditions réel) et publication du processus **W-6** (`AFTRSN-Budget-Tracker` `2mEakXhOGBs3dlyX` + `AFTRSN-W6.1 — Suivi Edition` `Qk0b1xwqkr0Pmtpk`). Version générique : `POS-GENERIQUE_Suivi-Budget-Sheets-Base_20260705.md`. Fiche technique : `Fiche_AFTRSN-Budget-Tracker_20260705.md`.

## 1. Prérequis AFTRSN

- **Template** `TEMPLATE_Event-Budget` (Google Sheets natif, `1VVGaXKJWpprnMMWwKPKJMHNClZZPWylHZ4DWwELfWEA`) dans `Departments/Accounting & Finance` (`1wwbZ5N7R_sctE8isJJWq4wCii_DBv_BJ`, compte **Hello**) — créé par conversion du xlsx réel de Karter (28.03 Mini Café), montants vidés, formules conservées.
- **API Google Sheets activée** sur le projet `909839129866` (faite le 05.07.2026 par Karter — passkey d'organisation requis, l'agent ne s'authentifie jamais).
- Credential **Drive HELLO** (`P7erICUpzKjuOV9E`) réutilisée pour Sheets : le scope Drive est accepté par l'API Sheets → **aucune credential Sheets dédiée nécessaire** (HTTP Request, `predefinedCredentialType: googleDriveOAuth2Api`).
- DB Notion 🎭 : Editions `dad7f826-…` (Status/Date/City/Capacity target/Lineup), Budget & Finance `05d7f280-…` (Line item/Category/Planned/Actual/Variance/Status/Edition).
- **AFTRSN-Comptable-Finance** (`F1wsXbsYZdEkwQ3n`) publié (DA-001 réglée le 05.07.2026), contrat `{query}` identique à Secretary.

## 2. Pattern construit (orchestrateur 7 nodes + sous-workflow 21 nodes)

**Orchestrateur** (cron `0 9 * * 5`, timezone Europe/Zurich) : Notion getAll Editions → Code « Editions a suivre » (hors Done/Cancelled, date requise ; date+3j passée → mode `closing`, sinon `weekly` ; 0 item = arrêt silencieux) → **Execute Workflow mode « each »** vers W6.1 (une exécution isolée par édition — références internes en `.first()` triviales) → Compter → Save Episode → Résumé.

**W6.1 par édition** : recherche Drive du fichier `YYYYMMDD_Event-Budget*` dans le dossier finance → absent ? **copie du template** (nom `YYYYMMDD_Event-Budget_<SlugEdition>`) → batchGet Sheets (2 onglets, `UNFORMATTED_VALUE`) → Code « Chiffres » (sous-totaux par ligne fixe, totaux, P&L, coût/revenu par tête via Capacity target, alertes ⚠️≥80 %/🔴≥100 %/🔴 hors budget, texte Telegram + query Comptable préparés ici) → query Notion des lignes de l'édition → upsert 9 lignes (5 catégories + 3 lignes revenus + P&L ; match par titre exact puis contains) → Comptable-Finance (executeOnce) → If clôture → update Edition `Status=Done` → Telegram (executeOnce) → Sortie.

**Choix structurants :**
- Le **détail vit dans Sheets** (travail humain), **Notion n'est que le résumé**, l'**agent analyse** — répartition dictée par Karter.
- Toute la logique par édition est dans le sous-workflow : l'orchestrateur ne fait que sélectionner et compter (règle « vue simple par étape », et ça évite la gymnastique d'index multi-branches).
- Message Telegram et query agent construits dans le node Code (une seule expression simple côté Telegram/Execute Workflow — moins de champs expression fragiles).
- Self-healing : une édition sans fichier budget en reçoit un automatiquement — P1 n'a pas eu à être modifié.

## 3. Difficultés rencontrées

1. Deux paramètres de requête `ranges` déclarés dans le node HTTP Request → n8n n'en a envoyé qu'un (sérialisation en objet, le dernier gagne) : le workflow lisait l'onglet Revenue à la place du Budget, **sans aucune erreur** — zéros partout.
2. Mes vérifications rapides `raw.includes('"NomDuNode"')` sur le payload d'exécution disaient « tous les nodes ont tourné » même pour des nodes jamais exécutés.
3. Publication de Comptable-Finance refusée : `409 There is a conflict with one of the webhooks`.
4. Appel du sous-workflow Comptable-Finance refusé à l'exécution : `Workflow is not active and cannot be executed`.
5. Premier test bloqué : API Sheets non activée sur le projet Google (`403 Forbidden`), et l'accès à la console Cloud exige un **passkey** que seul Karter peut fournir.
6. Édition de test laissée en `Confirmed` = elle serait retraitée (et re-notifiée) chaque vendredi.

## 4. Solutions implémentées

1. Les deux `ranges` passés **inline dans l'URL** (`…values:batchGet?ranges='Event%20Budget'!B1:F76&ranges='Event%20Revenue'!B1:J38&valueRenderOption=UNFORMATTED_VALUE`).
2. Décodage ciblé du format flatted **limité aux clés de `resultData.runData`** (le payload d'exécution embarque aussi le workflowData entier — les noms de nodes y figurent toujours).
3. `webhookId` du chatTrigger régénéré (UUID) avant activation — le workflow avait été dupliqué d'un autre agent et gardait son webhookId d'origine.
4. Publier le **callee avant** de tester l'appelant (sur cette instance, un sous-workflow doit être actif même pour un test manuel de l'appelant).
5. Activation de l'API par Karter (lien direct fourni) ; vérification par l'effet en relançant le test.
6. Données de test neutralisées **dans le statut que le process ignore** (`Cancelled`), suppression définitive laissée à Karter.

## 5. Leçons apprises

1. **Jamais de paramètre de requête dupliqué dans un node HTTP n8n** — n8n les sérialise en objet. Les répétitions vont dans l'URL. Le symptôme est silencieux : données plausibles mais fausses.
2. Chercher un nom de node dans le texte brut d'une exécution ne prouve rien : **seules les clés de `runData` font foi** (ou mieux : vérifier par les effets).
3. Un workflow **dupliqué** hérite des `webhookId` de l'original : les régénérer avant toute publication, sinon 409.
4. Chaîne d'activation : **sous-workflows d'abord** — y compris pour de simples tests manuels de l'appelant.
5. `POST /rest/workflows/<id>/run` (header `browser-id`) exécute un workflow depuis la console du navigateur connecté :**tests scriptables sans un seul clic UI** — plus fiable que le bouton Execute (leçon jumelle du « clic perdu » du 04.07). Marche aussi pour un appelant jetable qui teste un sous-workflow avec un payload contrôlé.
6. Le **mode « each » d'Execute Workflow** (v1.3) isole proprement chaque item dans une exécution du sous-workflow — validé en réel (2 éditions → 2 exécutions) ; c'est la réponse robuste au piège « sortie multi-items » du 04.07.
