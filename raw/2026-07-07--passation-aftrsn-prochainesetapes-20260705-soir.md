---
type: raw
title: "PASSATION_AFTRSN_ProchainesEtapes_20260705-SOIR"
source_url: "drive:1m64fJvzryjuVzijGolXV6J-rdNBg2vYc"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Passation — Prochaines étapes (05.07.2026 au soir)

**Statut :** Session du 05.07.2026 (après-midi/soir) clôturée. Ce document remplace `PASSATION_AFTRSN_ProchainesEtapes_20260705.md` comme relais courant vers la prochaine session.

---

## 1 · Boussole — toujours commencer ici

Avant toute action sur ce projet, lire **`FICHE_PROJET_AFTRSN_Automation.md`** (« Fiche Projet — AFTRSN Automation »). C'est le document compas : statut courant, deadlines, roadmap Étapes 1→4, règles métier actées, décisions ouvertes. Le présent document ne le remplace pas — il complète la transition de session et se lit en second. La Fiche Projet est à jour au 05.07.2026 au soir.

**Cap fixé par Karter (05.07.2026) : terminer d'abord la construction de TOUTE la plateforme, puis basculer en mode test/amélioration continue** — on ne s'arrête plus pour peaufiner, on finit W-4 et W-7, et ensuite chaque process s'améliore par la pratique.

> **Addendum session W-4 (05.07, plus tard le soir) :** DA-008 tranchée (périmètre v1 complet ; newsletter → fiche Notion ; Meta+Wix : accès fournis maintenant → v1.1). **W-4 Marketing-Planner cœur v1 construit, testé (3 chemins) et PUBLIÉ 🟢** (`AFTRSN-Marketing-Planner` `4qcia6aWt9ooQrle`, actif) : calendrier 11 créneaux J-21→J0 + copies IG (agent AFTRSN-Marketing) + newsletter en fiche Content & Social + Telegram + mémoire, **draft-only**, idempotence par relation, champ `Publish date` ajouté à Content & Social. Docs : `Fiche_AFTRSN-Marketing-Planner_20260705.md` + POS paire `POS-AFTRSN_Calendrier-Contenu-Agent-Notion_20260705.md` / `POS-GENERIQUE_Calendrier-Contenu-Agent-Base_20260705.md`. **P3 Meta Ads FAIT 🟢** : node `Meta - Campagne (PAUSE)` dans W-4 crée une campagne Meta en PAUSE par édition (testé de bout en bout), credential `facebookGraphApi` `3fAIGHeem6ebftQ3`, Ad Account `act_1927939831244305`, Page `694773763721517`. Setup Meta réalisé avec Karter (app **LUMINA OS** `992613030068111` publiée + utilisateur système **LUMINA HERMES N8N** `61591950411009` + **rôle « Gérer l'app »** = déblocage du token ; champ `is_adset_budget_sharing_enabled:false` obligatoire). **P4 Wix FAIT 🟢** via connecteur Wix (sans clé, couche agent/Hermès), brouillon 08.08 créé (`b5cb69a2-…`). **RESTANT : W-7 Knowledge-Capture** = dernier morceau plateforme. Données de test **nettoyées par l'agent** (36 pages Content & Social + 3 éditions archivées, 2 campagnes Meta de test supprimées) ; reste DA-009 = supprimer `ZZZ_20260705_tmp_w4_probe` (`Yk8t5R6WQxbr6tmB`).

**Échéances :** 18.07 = artistes du 08.08 confirmés (lieu déjà confirmé : Mini Café Bar ; artistes par WhatsApp, quasi bloqués — confirmations Karter en cours). **08.08 = run de validation A→Z** : Karter déroule lui-même tout le système de bout en bout.

## 2 · Ce qui a été fait cette session (05.07.2026 après-midi/soir)

1. **W-6 Budget-Tracker construit, testé, PUBLIÉ 🟢** (`AFTRSN-Budget-Tracker` `2mEakXhOGBs3dlyX` + `AFTRSN-W6.1 — Suivi Edition` `Qk0b1xwqkr0Pmtpk`, dossier `AFTRSN-PROCESS-05-Budget-Tracker`). Hebdo **vendredi 9h** Europe/Zurich ; le détail vit dans un **fichier Google Sheets par édition** (copie auto du template si absent), **Notion = résumé** (9 lignes Budget & Finance), seuils **⚠️ 80 % / 🔴 100 %**, coût/revenu **par tête** (capacité), **clôture auto J+3** (`Status=Done` + rapport), analyse rédigée par **Comptable-Finance**, Telegram, mémoire. **→ Étape 2 TERMINÉE.**
2. **Template budget créé** : `TEMPLATE_Event-Budget` (Sheets natif, `1VVGaXKJWpprnMMWwKPKJMHNClZZPWylHZ4DWwELfWEA`) dans `Departments/Accounting & Finance` (compte **Hello**), dérivé du xlsx réel de Karter (28.03 Mini Café, intact). Montants vidés, formules conservées, artistes → placeholders.
3. **DA-001 réglée : `AFTRSN-Comptable-Finance` PUBLIÉ 🟢** (webhookId du chatTrigger régénéré — conflit 409 d'un workflow dupliqué). Il est l'analyste budget officiel, appelé par W-6 comme les coordinators appellent Secretary.
4. **Brief artistes J-7 construit, testé, PUBLIÉ 🟢 (Artist-Coordinator v1.1)** — chaîne parallèle **additive** (12 nodes) sur le même déclencheur, la v1 outreach n'a pas bougé. Artistes `Confirmed` du `Lineup` d'une édition à ≤7 jours → brouillon brief (set time, rider, logistique — DE+EN) dans Gmail **Partners**, champ **`Brief J-7`** (ajouté à la base DJs/Artists) pour l'idempotence, Telegram 🎤, mémoire. Livré ~4 semaines avant l'échéance du 01.08.
5. **API Google Sheets activée** par Karter sur le projet `909839129866` (console protégée par passkey — l'agent ne s'authentifie jamais). La credential **Drive HELLO** suffit pour l'API Sheets (pas de credential Sheets dédiée).
6. **EP4 clôturée en conditions réelles** (décision Karter) : run orchestrateur multi-éditions validé — fichier `20260626_Event-Budget_EP4` créé (vide, à remplir a posteriori si chiffres disponibles), 9 lignes Notion, rapport Telegram, `Status=Done`.
7. **Correctif passation précédente :** le premier run réel des relances Venue-Coordinator (Maison 25, Soluna, The Palm Three) est attendu **lundi 06.07 à 9h** — la passation du matin disait « lundi 07.07 », or le 07.07 est un mardi ; les 3 fiches backfillées ont dépassé J+3 depuis longtemps, les brouillons partent au premier run ouvré.

## 3 · Fichiers produits / mis à jour cette session

| Fichier | Rôle |
|---|---|
| `Fiche_AFTRSN-Budget-Tracker_20260705.md` | Fiche technique W-6 (IDs, mapping template, précautions, nettoyages) |
| `POS-AFTRSN_Suivi-Budget-Sheets-Notion_20260705.md` + `POS-GENERIQUE_Suivi-Budget-Sheets-Base_20260705.md` | POS paire : suivi budget tableur + base résumé + agent, alertes seuils, clôture auto |
| `Fiche_AFTRSN-Artist-Coordinator_20260705.md` | Complétée : section **v1.1 brief J-7** (chaîne, prompt, idempotence, limites) |
| `AFTRSN_MarcheASuivre_Construction-n8n.md` | **§8 ajouté** (pièges du 05.07 au soir : query params dupliqués, runData vs texte brut, webhookId hérité, callee publié d'abord, `POST /rest/workflows/<id>/run`, mode « each », chaîne additive) |
| `FICHE_PROJET_AFTRSN_Automation.md` | Boussole, à jour au 05.07 au soir (Étape 2 terminée, DA-001 réglée, DA-007/DA-008 ouvertes) |
| `PASSATION_AFTRSN_ProchainesEtapes_20260705-SOIR.md` | Ce document |

## 4 · Prochaines étapes (priorité décroissante)

1. **Reprendre avec le périmètre du Marketing-Planner v1 (W-4)** — première action de la prochaine session. Déjà acté : **déclenchement à la demande, par édition** (contrat `{edition}`). À trancher (DA-008) : périmètre v1 (recommandation sur la table : calendrier 3 semaines dans **Content & Social 🎭** canonique `709b816f-…` + copies Instagram par l'agent **AFTRSN-Marketing** + newsletter draft + Telegram ; pubs Meta et page Wix en v1.1 dès accès API fournis) ; **destination de la newsletter draft** (brouillon Gmail Hello / fiche Notion / les deux) ; prévoir d'ajouter un champ date (`Publish date`) à Content & Social. ⚠️ Doublon de base : toujours **By ID**.
2. **Construire W-7 Knowledge-Capture** (P5 : ingestion debriefs → `aftrsn_memory` collection `insights` ; le volet closing budget est déjà couvert par W-6) — dernier morceau de la plateforme.
3. **Lundi 06.07, 9h — premier run réel Venue-Coordinator** : 3 brouillons de relance attendus (fils Partners) + 3 notifications Telegram. Karter relit et envoie. Vendredi 10.07, 9h — **premier run planifié du Budget-Tracker** (traitera les éditions ouvertes du moment).
4. **Artistes 08.08 (Karter, manuel)** : confirmations WhatsApp + choix des résidents ; dès confirmation, consigner chaque artiste dans DJs/Artists (**email + statut `Confirmed`**) et le lier à l'édition via **`Lineup`** — sinon le brief J-7 automatique ne partira pas (limite connue : pas d'email = pas de brief, canal WhatsApp différé).
5. **Préparer le run A→Z du 08.08** : remettre le payload épinglé du trigger P1 sur les vraies données EP5 ; l'édition créée déclenchera naturellement budget (vendredi) et brief (01.08).
6. **Pousser au canon** (`wiki/`, collection `canon`) : « from day to night » (prioritaire), décisions LUMINA OS du 04.07, rangement n8n, POS génériques (dont la nouvelle paire budget). Un document de session ne corrige pas la connaissance de fond (SYSTEM-CANON §5).
7. **Décisions Karter en attente** : DA-001b (Qualite-Compliance), DA-002, DA-004, DA-006, **DA-007** (`ZZZ_20260705_tmp_budget_test`), **DA-008** (périmètre W-4 + accès Meta/Wix).
8. **Nettoyages Karter** : brouillons ZZTEST Gmail Partners (3 Venue + 3 Artist + 1 brief J-7) ; fiches Notion ZZTEST (`ZZTEST-Venue-Coordinator`, `ZZTEST-Artist-Coordinator`, `ZZTEST-Budget-Tracker` + 9 lignes budget, `ZZTEST-Brief-Artist`, `ZZTEST-Brief-Edition` — toutes neutralisées) → corbeille ; fichier Drive `20260720_Event-Budget_ZZTESTBudgetTracker` → corbeille ; reconfirmer les 6 emails ⚠️ avant envoi (Rimini, Stazione Paradiso, Primitivo, Frieda's Büxe, Kauz, Longstreet) ; remplir le fichier budget EP4 si les chiffres réels existent.
9. **Surveiller en production** : Email-Responder (qualité des drafts, volume) ; premiers runs réels Venue-Coordinator (lundi) et Budget-Tracker (vendredi).

## 5 · Protocole permanent — marches à suivre et POS (instruction pour toutes les prochaines sessions)

**Après chaque étape validée**, toute explication réutilisable produite pendant le travail (mode d'emploi pratique généralisable, ou procédure technique détaillée vérifiée) doit être **automatiquement extraite en fichier(s) `.md` et sauvegardée dans `/Users/karter/Claude/Projects/AFTRSN Automation/`** — jamais laissée uniquement dans la conversation ou noyée dans un récap.

**Deux types de document, deux conventions de nommage :**
- **Marche à suivre** — mode d'emploi pratique, réutilisable, **sans date** (document vivant, mis à jour sur place) : `<Scope>_MarcheASuivre_<Sujet-en-Kebab-Case>.md`, `Scope` = `AFTRSN` ou `Generique`.
- **POS** (Procédure Opérationnelle Standardisée) — procédure technique détaillée, snapshot **daté** d'un état vérifié : `POS-<SCOPE>_<Sujet-en-Kebab-Case>_<AAAAMMJJ>.md`, `SCOPE` = `GENERIQUE` / `AFTRSN` / `LUMINA`.

**Règle de la paire systématique :** chaque document **spécifique AFTRSN** (marche à suivre ou POS) est produit avec **une seconde version générique** en parallèle, débarrassée des éléments propres à AFTRSN (IDs, comptes, noms de bases), réutilisable pour toute autre marque gérée via LUMINA OS. Si la procédure est générique par nature, un seul document `Generique_`/`POS-GENERIQUE` (ou `POS-LUMINA` si elle concerne l'architecture LUMINA OS) suffit — pas de doublon artificiel.

**Sections obligatoires dans chaque document produit :**
- **Difficultés rencontrées** — les blocages, erreurs et ambiguïtés effectivement rencontrés (pas hypothétiques).
- **Solutions implémentées** — ce qui a concrètement résolu chaque difficulté (diagnostic exact + correctif appliqué).
- **Leçons apprises** — le principe généralisable, formulé pour rester utile sans se souvenir de l'incident d'origine.

**Règles d'application :**
1. **Détection automatique** — dès qu'une explication étape-par-étape émerge (réponse à une question ou sous-produit d'un débogage), évaluer sa réutilisabilité et l'extraire si oui, **sans attendre la fin de session**.
2. **Sauvegarde effective** dans le dossier projet (jamais seulement dans le scratchpad).
3. **Référencement obligatoire** — chaque nouveau document (y compris sa paire générique) est ajouté à « ✅ Déjà fait › Architecture & documentation » de `FICHE_PROJET_AFTRSN_Automation.md` au moment de sa création, et à la table §3 de la passation courante.
4. **Portée LUMINA OS vs métier AFTRSN** — si le contenu concerne l'architecture/gouvernance du système LUMINA OS lui-même, signaler à Karter qu'il mérite d'être poussé vers le dépôt canonique (`wiki/` → collection `canon`) — un document de session seul ne corrige jamais durablement la connaissance de fond (SYSTEM-CANON §5). Cas acté le 05.07 : un **skill opérationnel** peut être appris directement à Hermès (voir POS dédiée) — Hermès est l'exécutant et l'apprenant, les spécialistes guident.
5. **Présentation à l'utilisateur** — chaque document finalisé est présenté via le mécanisme standard (`present_files`), pas seulement décrit en texte.

## 6 · Difficultés rencontrées, solutions implémentées, leçons apprises — session du 05.07.2026 (soir)

| # | Difficulté | Solution | Leçon |
|---|---|---|---|
| 1 | batchGet Sheets : deux paramètres `ranges` déclarés, un seul envoyé — lecture silencieusement fausse (zéros plausibles, aucun message d'erreur) | Ranges **inline dans l'URL** encodée | n8n sérialise les query params en objet : jamais de paramètre dupliqué dans un node HTTP ; le symptôme est des données plausibles mais fausses |
| 2 | Vérifications rapides `raw.includes('"NomDuNode"')` toutes positives même pour des nodes jamais exécutés | Décodage flatted ciblé sur les **clés de `resultData.runData`** | Le payload d'exécution contient aussi le workflowData : chercher un nom de node dans le texte brut ne prouve rien |
| 3 | Publication Comptable-Finance : 409 « conflict with one of the webhooks » | `webhookId` du chatTrigger régénéré (UUID) puis activation | Un workflow dupliqué hérite des webhookIds de l'original — les régénérer avant toute publication |
| 4 | Appel d'un sous-workflow non publié : « Workflow is not active and cannot be executed » | Publier le callee avant de tester l'appelant | Ordre d'activation : sous-workflows d'abord, y compris pour de simples tests manuels |
| 5 | API Sheets non activée (403) ; console Google Cloud verrouillée par **passkey** | Activation par Karter (lien direct) ; vérification par l'effet en relançant le test | Toute nouvelle API Google = prévoir l'activation projet et l'aller-retour humain ; l'agent ne s'authentifie jamais (passkey/2FA = humain uniquement) |
| 6 | Tests UI fragiles (boutons Execute, références DOM) | **`POST /rest/workflows/<id>/run`** + appelant jetable re-PATCHé par cas de test + lecture de l'exécution par id | Toute la boucle build→test→fix est scriptable par l'API interne — zéro clic UI, reproductible |
| 7 | Étendre un workflow publié (Artist-Coordinator) sans re-tester ses 4 chemins | **Chaîne parallèle additive** sur le même trigger (v1 intacte), puis re-publication | Une extension additive vaut mieux qu'une modification du node de sélection : le périmètre de test se limite au nouveau chemin |
| 8 | Édition de test en statut vivant = retraitée à chaque run planifié (bruit hebdo) | Fiches ZZTEST neutralisées en `Cancelled`, suppression laissée à Karter | Neutraliser les données de test **dans le statut que le process ignore** — la corbeille reste une décision humaine |
| 9 | `jsCode` d'un node existant illisible via l'API navigateur (filtre « sensitive », faux positif) | Contrat reconstruit depuis les nodes lisibles (Switch, Gmail, Telegram) + POS | Les POS et le nommage clair des nodes sont aussi une **assurance-lecture** : quand le code source est inaccessible, le contrat documenté suffit |

---
*Document rédigé en fin de session le 05.07.2026 au soir.*
