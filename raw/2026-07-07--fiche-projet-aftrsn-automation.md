---
type: raw
title: "FICHE_PROJET_AFTRSN_Automation"
source_url: "drive:1K246rQVK-kWqQRTMiAXR0cJo42Q8sHXe"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche Projet — AFTRSN Automation

**Dernière mise à jour :** 06.07.2026 (session W-7 Knowledge-Capture — **PLATEFORME COMPLÈTE**, Étapes 1→4 terminées ; incident credential Postgres résolu + health-check mémoire ajouté)
**Objectif global :** Système d'automatisation n8n gérant le cycle de vie complet d'une édition AFTER SUN, de la prospection venue/artistes jusqu'au closing budgétaire et à la capture de connaissance post-event.
**Prochain événement (deadline MVP) :** 08.08.2026
**Deadline critique :** Venue + minimum 2 artistes confirmés avant le **18 juillet 2026** (J-21) — en dessous, la fenêtre promo passe de 3 à 2 semaines, risque direct sur la fréquentation.

---

## ✅ Déjà fait

### Architecture & documentation
- [x] Document de passation architecture (`PASSATION_AFTRSN_AutomationArchitecture_20260702.md`)
- [x] Flowchart détaillé node-par-node de W-1 (`W1_Event-Creator_Flowchart_Spec_20260702.md`), réconcilié avec le Notion réel
- [x] Brand Voice générique AFTRSN (`AFTRSN_Brand_Voice_Generic.md`)
- [x] SOP / Runbook Event Creation W-1 (`AFTRSN_SOP_Event-Creation_W1.md`)
- [x] Marche à suivre AFTRSN pour construction n8n (`AFTRSN_MarcheASuivre_Construction-n8n.md`)
- [x] Marche à suivre générique construction n8n via navigateur (`Generique_MarcheASuivre_Construction-n8n-Navigateur.md`)
- [x] Récap de session 03.07.2026 (`RECAP_Session_20260703_W1-n8n-Build.md`)
- [x] Fiche technique complète W-1 (`Fiche_AFTRSN-Event-Creator_20260703.md`)
- [x] Draft découpage sous-workflows W-1 (`W1_Decoupage_Sous-Workflows_20260703.md`) — **❌ remplacé le 04.07.2026** par l'architecture par processus métier ; conservé pour référence
- [x] **Architecture par processus métier** (`AFTRSN_Architecture_Processus_20260704.md`) — chaîne P1 Création d'édition → P5 Post-event + processus transverses, draft v1 en attente de validation Karter
- [x] Clarification Hermès (`Hermes_Clarification_20260703.md`)
- [x] Passation prochaines étapes (`PASSATION_AFTRSN_ProchainesEtapes_20260703.md`)
- [x] Passation prochaines étapes 04.07 (`PASSATION_AFTRSN_ProchainesEtapes_20260704.md`) — **relais courant**
- [x] Fiche technique Email-Responder (`Fiche_AFTRSN-Email-Responder_20260704.md`)
- [x] POS découpage orchestrateur/sous-workflows via API n8n (`POS-GENERIQUE_Decoupage-Workflow-n8n-Orchestrateur-Sous-Workflows_20260704.md`) — générique par nature, pas de paire
- [x] POS répondeur email à drafts via agent (`POS-AFTRSN_Repondeur-Email-Drafts-Agent_20260704.md` + paire `POS-GENERIQUE_Repondeur-Email-Drafts-Agent_20260704.md`)
- [x] Architecture par processus métier (`AFTRSN_Architecture_Processus_20260704.md`) — chaîne P1→P5, transverses, règle « Maestro délègue, Hermès exécute » en option (b) avec cycle de supervision Maestro ↔ Hermès (critères d'acceptation, rapport de skills, banque de données, boucle d'amélioration) ; décisions Karter du 04.07.2026
- [x] Fiche technique Venue-Coordinator (`Fiche_AFTRSN-Venue-Coordinator_20260704.md`)
- [x] POS outreach + relances piloté par Notion, draft-only (`POS-AFTRSN_Outreach-Relances-Notion-Drafts_20260704.md` + paire `POS-GENERIQUE_Outreach-Relances-Base-Drafts_20260704.md`)
- [x] Marche à suivre construction n8n : §7 ajouté (format simplifié Notion `property_*`, vérification par les effets, timezone explicite + re-publication, credentials via nodes existants)
- [x] Fiche technique Artist-Coordinator (`Fiche_AFTRSN-Artist-Coordinator_20260705.md`)
- [x] POS apprendre un skill à Hermès (`POS-LUMINA_Apprendre-Un-Skill-A-Hermes_20260705.md`) — générique par nature (architecture LUMINA OS), pas de paire
- [x] POS backfill outreach manuel (`POS-AFTRSN_Backfill-Outreach-Manuel_20260705.md` + paire `POS-GENERIQUE_Backfill-Outreach-Manuel_20260705.md`)
- [x] Fiche technique Budget-Tracker (`Fiche_AFTRSN-Budget-Tracker_20260705.md`) — 05.07 au soir
- [x] POS suivi budget Sheets + Notion (`POS-AFTRSN_Suivi-Budget-Sheets-Notion_20260705.md` + paire `POS-GENERIQUE_Suivi-Budget-Sheets-Base_20260705.md`) — 05.07 au soir
- [x] Fiche Artist-Coordinator complétée v1.1 (section brief J-7) — 05.07 au soir
- [x] Marche à suivre construction n8n : **§8 ajouté** (query params dupliqués, runData vs texte brut, webhookId hérité, callee publié d'abord, `POST /rest/workflows/<id>/run`, mode « each », chaîne parallèle additive) — 05.07 au soir
- [x] Passation prochaines étapes 05.07 au soir (`PASSATION_AFTRSN_ProchainesEtapes_20260705-SOIR.md`) — remplacée
- [x] Passation prochaines étapes 05.07 nuit (`PASSATION_AFTRSN_ProchainesEtapes_20260705-NUIT.md`) — **relais courant** (prochaine étape = W-7 Knowledge-Capture)
- [x] Fiche technique Marketing-Planner W-4 (`Fiche_AFTRSN-Marketing-Planner_20260705.md`) — 05.07 au soir (cœur v1)
- [x] POS générateur de calendrier de contenu (agent → Notion, draft-only) (`POS-AFTRSN_Calendrier-Contenu-Agent-Notion_20260705.md` + paire `POS-GENERIQUE_Calendrier-Contenu-Agent-Base_20260705.md`) — 05.07 au soir
- [x] POS token System User Meta Ads pour n8n (`POS-GENERIQUE_Meta-Ads-SystemUser-Token-n8n_20260705.md`) — 05.07 au soir (générique par nature, pas de paire) ; **à pousser au canon** (procédure LUMINA OS réutilisable)
- [x] Spec W-7 Knowledge-Capture (`Spec_AFTRSN-Knowledge-Capture_W7_20260705.md`) — validée Karter puis construite
- [x] Fiche technique W-7 Knowledge-Capture (`Fiche_AFTRSN-Knowledge-Capture_20260706.md`) — 06.07 (publié)
- [x] POS capture de connaissance débrief (`POS-AFTRSN_Capture-Connaissance-Debrief_20260706.md` + paire `POS-GENERIQUE_Capture-Connaissance-Debrief_20260706.md`) — 06.07
- [x] **POS réparation credential Postgres partagée** (`POS-LUMINA_Reparation-Credential-Postgres-Partagee_20260706.md`) — 06.07 (runbook infra LUMINA) ; **à pousser au canon**

### Notion — "The Backstage"
- [x] Reconstruction de The Backstage dans le nouveau teamspace
- [x] Création des 9 DB Operations (Editions, Venues, DJs/Artists, Budget & Finance, Content & Social, Marketing & KPI, Partners & Sponsors, Media Assets, Cities/Expansion)
- [x] Migration des données réelles vers les nouvelles DB
- [x] Recréation de Human Space, Agents, Brand Source of Truth

### W-1 · AFTRSN-Event-Creator (n8n)
- [x] Credential Google OAuth Drive créée et testée
- [x] Credential Google Calendar créée (réutilisation du client OAuth Drive), calendrier `hello@aftersunpeople.com`
- [x] Node 1–2 : Trigger Maestro + validation input (`If`)
- [x] Node 3 : Normalisation & slug (`Edit Fields`) — bugs de dates corrigés (campaign_start, artist_brief, media_date_prefix)
- [x] Node 4–5 : Dossier Drive édition + arborescence média complète (Photos/Videos × Raw/Edited, format canon)
- [x] Node 6 : Création event Google Calendar
- [x] Node 7–8 : Upsert Venue + création page Edition dans Notion
- [x] Node 9 : Injection de la checklist (8 sections, 27 to-dos) dans le corps de la page Edition — **construit, pas encore testé en exécution réelle**
- [x] Node 10 : Amorçage des 5 lignes Budget & Finance (Venue/Artists/Production/Marketing/Media)
- [x] Node Merge : synchronisation des branches Calendar + Budget
- [x] Node 11 : Sauvegarde épisode mémoire (`LUMINA-MEMORY-WRITE/WEBHOOK`)
- [x] Node 12 : Agrégation du résumé final (`status, notion_url, drive_url, calendar_url`)
- [x] Node 13 : Branche d'erreur (`status:"error", step, message`), adaptée en node Set (pas de "Respond to Webhook" — trigger réel = Execute Workflow)
- [x] Tous les nodes renommés clairement

---

## ⏳ Reste à faire

### W-1 / P1 — Finalisation (avant de passer à la suite)
- [x] **04.07.2026 — Correctif « Create an event »** (option B : description → fiche Notion), suppression `Merge2`, repointage des 4 nodes Notion vers les DB 🎭 (teamspace AFTER SUN PEOPLE - HQ, canonique), suppression des doublons de test (Venue + EP5, corbeille Notion)
- [x] **04.07.2026 — Découpage en orchestrateur + 6 sous-workflows métier** (AFTRSN-P1.1 → P1.6, voir `AFTRSN_Architecture_Processus_20260704.md`) ; mécanique validée en exécution ; bug latent `media_date_prefix` corrigé au passage
- [x] Partage Notion 🎭 ↔ AFTRN-n8n (fait par Karter le 04.07)
- [x] **04.07.2026 — TEST END-TO-END VALIDÉ ✅** : chaîne complète en ~15 s, résumé final avec URLs réelles. 4 bugs hérités corrigés en route : AOD manquant sur `Chercher Venue` (0 résultat = chaîne morte silencieuse), `Verifier` testait `items.length` au lieu de la présence d'un `id`, expressions malformées `Venue - Nouvelle` + `Agreger resume` (sans `=` ni `}}`), sortie multi-items de P1.2 (executeOnce ajouté + refs `.first()` dans l'orchestrateur)
- [x] Nettoyage Notion 🎭 : 3 éditions ZZTEST + 15 lignes budget + 1 venue → corbeille
- [x] Nettoyage Drive fait par Karter (4 dossiers ZZTEST + anciens dossiers `_` du 03.07) ; **convention renommée `AFTSN` → `AFTRSN`** (dossiers existants renommés, P1.1 `media_date_prefix` + checklist P1.3 + template Notion alignés)
- [x] Nettoyage Calendar fait par Karter (3 events ZZTEST 30.09.2026) — **toutes les données de test sont purgées**
- [x] **04.07.2026 — PUBLIÉ ✅** : orchestrateur + 6 sous-workflows publiés (« v1.0 — P1 Event-Creation »), tags passés en 🌞 AFTRSN · 🔄 automation · 🟢 PROD. **P1 est en production.** (Rappel : remettre le payload épinglé du trigger sur les vraies données EP5 avant le premier run réel.)
- [ ] Tester le node **Notion — Injecter checklist** en exécution réelle (jamais lancé pour éviter une cascade de doublons de test)
- [ ] Lancer un test complet de bout en bout avec un vrai payload Maestro (`edition_name`, `city`, `event_date`, `venue_name`, `capacity_target`)
- [ ] Nettoyer manuellement les dossiers Drive `_` dupliqués créés pendant les tests (plusieurs occurrences — à faire par Karter, non supprimés automatiquement)
- [ ] Publier ("Publish") le workflow dans n8n une fois validé
- [ ] (Optionnel, cosmétique) Corriger les caractères `->`/`-` de la checklist injectée en `→`/`—` pour coller exactement au template Notion d'origine

### Étape 1 — Semaine 1 (02–08 juillet) · Fondations
- [x] Credential Google OAuth (Drive + Calendar) — Gmail AFTRSN + Sheets restent à créer/tester
- [x] W-1 Event-Creator construit (finalisation ci-dessus en cours)
- [x] **04.07.2026 — Email-Responder construit, testé et PUBLIÉ 🟢** (`AFTRSN-Email-Responder`, dossier `AFTRSN-PROCESS-02-Email-Responder`, fiche : `Fiche_AFTRSN-Email-Responder_20260704.md`). Lecture Gmail horaire → drafts via Secretary → notification Telegram détaillée (sans codes techniques) → mémoire. Jamais d'envoi automatique. Credential Gmail créée par Karter, API Gmail activée (projet `909839129866`), chat_id Telegram 776345147 (trouvé via @rawdatabot). **→ Étape 1 de la roadmap TERMINÉE.**

### Étape 2 — Semaine 2 (09–15 juillet) · Coordination
- [x] **04.07.2026 — Venue-Coordinator construit, testé (4 chemins) et PUBLIÉ 🟢 avec 5 jours d'avance** (`AFTRSN-Venue-Coordinator` `Su9tO0fS39u9itlU`, dossier `AFTRSN-PROCESS-03-Venue-Coordinator`, fiche : `Fiche_AFTRSN-Venue-Coordinator_20260704.md`). Décisions Karter : tracking dans la base Venues 🎭 (pas Sheets), compte **Partners** (credential `PARTNERS-GMAIL` créée le 04.07), tout en brouillon (premier contact ET relances), allemand + version anglaise, jours ouvrés 9h (Europe/Zurich). Base Venues enrichie : champs `Email` + `Gmail Thread`. Arrêt des relances = passage manuel du Status à `In discussion`.
- [x] **05.07.2026 — Backfill de l'outreach manuel pré-automatisation** : 3 lieux contactés depuis partners@aftersunpeople.com sans réponse, retrouvés dans les envoyés du compte Partners (via workflow temporaire `ZZZ_20260705_tmp_partners_sent_lookup`) et consignés en `To contact` avec date réelle + fil d'origine : **Maison 25** (info@urbanagency.ch, 16.06, proposition de résidence mensuelle), **Soluna** (contact@soluna-zurich.ch, 12.05), **The Palm Three** (hello@thepalmthree.ch, 12.05). → Relances automatiques dans les fils d'origine dès lundi 07.07, 9h. Correctif v1.2.1/v1.0.2 au passage : la seconde relance part 4 jours après la première (et plus 7 jours après le premier email) — évitait deux relances consécutives sur les lieux backfillés.
- [x] **05.07.2026 — Situation artistes 08.08 clarifiée (Karter)** : le booking se gère **par téléphone/WhatsApp, pas par email** — plusieurs artistes déjà contactés et intéressés (confirmations en attente), plus les **DJ résidents** parmi lesquels choisir 2–3 accompagnants. **Le 08.08 est quasi bloqué.** L'Artist-Coordinator (email) servira aux bookings futurs (artistes hors réseau direct) ; pas d'automatisation WhatsApp (décision antérieure : canal WhatsApp différé). ⚠️ La deadline du 18.07 est en voie de résolution manuelle — il reste à **confirmer** les artistes pressentis.
- [x] **05.07.2026 — Le 08.08 = run de validation A→Z (décision Karter)** : Karter réserve l'édition du 08.08 pour dérouler lui-même tout le système de bout en bout (création d'édition → promo → post-event) et observer comment ça se passe. Les processus publiés doivent être prêts et documentés pour ce test grandeur nature.
- [x] **05.07.2026 — Déclenchement par statut « To contact » (règle métier Karter, v1.2/v1.0.1 republiées)** : `Prospect` = réserve silencieuse, rien ne part ; c'est Karter qui pose `To contact` sur un lieu/artiste → premier contact + relances. Le rythme (~2–3 premiers contacts lieux/semaine) est une **décision humaine volontairement hors process**. Statut ajouté aux deux bases (Venues + DJs/Artists).
- [x] **05.07.2026 — Calendrier lieux clarifié (Karter)** : pas d'urgence — le lieu actuel (Mini Café Bar) est en place ; un **nouveau lieu éventuellement à partir d'octobre**, selon les conditions offertes par le lieu actuel. Août–septembre : recherche d'un lieu **réputé en musique électronique** + promotion agressive. La deadline du 18.07 reste centrée sur les 2 artistes du 08.08.
- [x] **05.07.2026 — Concept « from day to night » acté (règle métier Karter)** : AFTER SUN n'est pas cantonné au day party — en Europe, ~2 mois d'été favorables au jour, le reste bascule vers la nuit (précédents Londres/Turquie en club, même nom, même DA Afro House). Formats : 15h–22h / 19h–02h / 22h–04h/06h selon lieu et saison. Conséquences : **10 clubs zürichois ajoutés** à la base Venues (Zukunft, Hive, Supermarket, Frieda's Büxe, Kauz, Exil, Kanzlei, Kaufleuten, Zentralwäscherei, Longstreet ; ⚠️ emails à reconfirmer : Frieda's Büxe, Kauz, Longstreet) ; Bar 3000 fusionné avec Club Zukunft (même contact) ; **prompt Venue-Coordinator v1.1 republié**. ⚠️ Cette règle métier est une connaissance de marque de fond → **à pousser au canon** (`aftrsn_memory`, collection `canon`).
- [x] **04.07.2026 — 12 lieux prospects créés dans la base Venues** (recherche web validée par Karter : Kasheme, Barfussbar, Bar 3000, Rimini, Stazione Paradiso, Primitivo, Chuchi am Wasser, Frau Gerolds, El Lokal, Photobastei, Ziegel oh Lac, Mama Shelter ; 3 emails ⚠️ à reconfirmer avant envoi : Rimini, Stazione Paradiso, Primitivo). **Le lieu du 08.08 est confirmé (Mini Café Bar)** → la deadline du 18.07 porte surtout sur les 2 artistes.
- [x] **05.07.2026 — Artist-Coordinator construit, testé (4 chemins) et PUBLIÉ 🟢** (`AFTRSN-Artist-Coordinator` `VpTCflWHXkU49LDB`, dossier `AFTRSN-PROCESS-04-Artist-Coordinator`, fiche : `Fiche_AFTRSN-Artist-Coordinator_20260705.md`). Même pattern que Venue-Coordinator ; base DJs/Artists alignée (champs Email + dates + Gmail Thread ajoutés) ; prompt booking avec contexte 08.08 Mini Café Bar + genres Afro House. **Brief J-7 reporté en v1.1** (décision Karter). ⚠️ Base vide : Karter doit créer ses cibles de booking (nom + email, statut Prospect) — c'est le chemin critique du 18.07.
- [x] **05.07.2026 au soir — v1.1 Artist-Coordinator : brief artistes J-7 construit, testé et PUBLIÉ 🟢 avec ~4 semaines d'avance** (chaîne parallèle additive de 12 nodes, la v1 outreach n'a pas bougé). Artistes `Confirmed` du `Lineup` d'une édition à ≤7 jours → brouillon de brief (set time, rider, logistique — allemand + anglais) dans Gmail **Partners** (nouveau fil), champ **`Brief J-7`** (ajouté à la base DJs/Artists) daté pour l'idempotence, Telegram 🎤, épisode mémoire. Testé : génération + idempotence (re-run = 0 brief) + chaîne v1 muette. ⚠️ Le brief ne part que si l'artiste est lié à l'édition via `Lineup`, en `Confirmed`, **avec un email** → consigner les artistes du 08.08 dès confirmation WhatsApp.
- [x] **05.07.2026 au soir — W-6 Budget-Tracker construit, testé et PUBLIÉ 🟢 avec ~1 semaine d'avance** (`AFTRSN-Budget-Tracker` `2mEakXhOGBs3dlyX` + `AFTRSN-W6.1 — Suivi Edition` `Qk0b1xwqkr0Pmtpk`, dossier `AFTRSN-PROCESS-05-Budget-Tracker`, fiche : `Fiche_AFTRSN-Budget-Tracker_20260705.md`). Décisions Karter : **hebdo vendredi 9h** (Europe/Zurich), seuils **80 %/100 %**, **clôture auto J+3** (`Status=Done`), le détail vit dans le **fichier Sheets par édition** (template `TEMPLATE_Event-Budget` créé depuis le xlsx réel de Karter, dossier `Accounting & Finance` compte Hello), **Notion = résumé** (9 lignes Budget & Finance par édition : 5 catégories + 3 revenus + P&L), **Comptable-Finance = analyste** (coût/revenu par tête vs capacité, dérives, recommandation). Fichier budget créé automatiquement du template s'il manque. API Google Sheets activée (05.07). **EP4 clôturée en test réel** (fichier `20260626_Event-Budget_EP4` créé vide, à remplir a posteriori). **→ Étape 2 de la roadmap TERMINÉE (le 05.07 au soir, 4 jours avant son début théorique).**
- [x] **Décision DA-001 réglée le 05.07.2026 par l'usage** : `AFTRSN-Comptable-Finance` **publié** (« v1.0 — Comptable-Finance », webhookId du chatTrigger régénéré pour lever un conflit 409) — il est l'analyste budget officiel appelé par W-6. `AFTRSNQualite-Compliance` reste non publié (décision encore ouverte, voir DA-001b).
- [ ] Nettoyage post-test Artist-Coordinator (Karter) : 3 brouillons ZZTEST dans Gmail Partners + fiche `ZZTEST-Artist-Coordinator` à la corbeille
- [ ] Nettoyage post-test Venue-Coordinator (Karter) : supprimer les 3 brouillons de test dans Gmail Partners + mettre la fiche `ZZTEST-Venue-Coordinator` à la corbeille Notion (neutralisée)
- [ ] Nettoyage post-test Budget-Tracker + brief J-7 (Karter) : fiches Notion `ZZTEST-Budget-Tracker` (+ ses 9 lignes budget), `ZZTEST-Brief-Artist`, `ZZTEST-Brief-Edition` (toutes neutralisées `Cancelled`) → corbeille ; fichier Drive `20260720_Event-Budget_ZZTESTBudgetTracker` → corbeille ; 1 brouillon Gmail Partners « Artist brief J-7 » (ZZTEST) → supprimer

### Étape 3 — Semaine 3 (16–22 juillet) · Marketing
- [x] **05.07.2026 au soir — W-4 Marketing-Planner cœur v1 construit, testé (3 chemins) et PUBLIÉ 🟢 avec ~11 jours d'avance** (`AFTRSN-Marketing-Planner` `4qcia6aWt9ooQrle`, actif). Déclenchement **à la demande, par édition** (`{edition}`). Lecture fiche édition (Editions `dad7f826`) → calendrier 11 créneaux J-21→J0 + newsletter → rédaction par l'agent **AFTRSN-Marketing** (`wyUEojwZjSKFRaRL`, input `query`/sortie `output`) → **pages brouillon dans Content & Social 🎭** (canonique `709b816f`, By ID ; **champ `Publish date` ajouté** ; copie dans le corps de page) → **newsletter en fiche Notion** (Channel `Other`, Owner `Comms`) → Telegram → mémoire (`episodic`). **Idempotence par relation** (2e run = 0 création). Draft-only. Testé sur `ZZTEST-Marketing-Planner` (12 pages OK), idempotence, et édition introuvable. Fiche : `Fiche_AFTRSN-Marketing-Planner_20260705.md`. **DA-008 tranchée** (voir ci-dessous).
- [x] **05.07.2026 au soir — W-4 P3 Meta Ads construit, testé de bout en bout et PUBLIÉ 🟢** : node `Meta - Campagne (PAUSE)` (onError=continue) crée une **campagne Meta en PAUSE** par édition (`act_1927939831244305`, Page `694773763721517`), credential n8n `facebookGraphApi` `3fAIGHeem6ebftQ3`. Jamais publiée (revue Ads Manager). Mise en place Meta faite : app **LUMINA OS** publiée + utilisateur système **LUMINA HERMES N8N** avec rôle « Gérer l'app » + token System User « Jamais ». Corps requis : ajouter `is_adset_budget_sharing_enabled:false`. Adsets/creatives restent manuels. Données de test nettoyées (36 pages CS + 3 éditions archivées, 2 campagnes test supprimées).
- [x] **05.07.2026 — P4 Wix résolu (couche agent, sans clé)** : page événement créée via le **connecteur Wix** (Events V3, `CreateEvent draft:true`, RSVP sans-billet), **pas dans n8n** (n8n ne peut pas appeler le connecteur → action Hermès/agent). Brouillon de démo 08.08 créé (`b5cb69a2-…`). Karter n'a **aucune clé Wix** à fournir.
- [ ] Lancer la campagne officielle 08.08 (annonce Instagram + newsletter)

### Étape 4 — Post-event (août) · Clôture
- [x] **06.07.2026 — W-7 Knowledge-Capture construit, testé (Phase A+B) et PUBLIÉ 🟢** (`AFTRSN-Knowledge-Capture` `AGoADrWzpZDa5b1g`, cron quotidien 09h). Modèle deux temps : Phase A ouvre une fiche débrief (base Notion **Post-Event Débriefs** `21f26092-…`) auto quand une édition passe `Done` ; Phase B capture les insights dans `aftrsn_memory`/`insights` quand Karter passe la fiche en `Prêt`. Gate humain. Fiche : `Fiche_AFTRSN-Knowledge-Capture_20260706.md`. **→ PLATEFORME AFTRSN COMPLÈTE (Étapes 1→4 terminées).**
- [x] Closing budget déjà couvert par W-6 (Budget-Tracker) ; W-7 = capture de connaissance.

### Incident & durcissement infra (06.07.2026)
- [x] **Panne credential Postgres résolue** : la credential partagée `Ojp0aTYdIFfNsr4u` (« Postgres account ») est devenue fantôme (listée mais illisible) → tout le cerveau down (mémoire + 17 agents/workflows actifs). Karter a recréé **`LUMINA_Postgres` `mlN3noHH3TT9C6Eu`** (host = conteneur `postgresql-…`, base `n8n`) ; l'agent a repointé les 24 nodes + **republié** les 17 actifs (piège : `/activate` obligatoire, un PATCH ne touche que le brouillon). Vérifié write+read+Maestro. Runbook : `POS-LUMINA_Reparation-Credential-Postgres-Partagee_20260706.md`.
- [x] **Health-check mémoire ajouté** : `LUMINA-HEALTHCHECK-MEMOIRE` (`M4rvTAqAlhJUzNSy`, actif, 07h30 quotidien) ping le retrieval et alerte Telegram si la mémoire retombe.

### Décisions ouvertes (Karter)
- [x] ~~**DA-001**~~ — **réglée pour Comptable-Finance le 05.07.2026 : publié**, analyste budget officiel de W-6 (voir Étape 2)
- [ ] **DA-001b** — `AFTRSNQualite-Compliance` : câblé mais non publié, décider de publier ou documenter la raison
- [ ] **DA-002** — Supprimer `ZZZ_20260702_tmp_hermes_reach_test`
- [ ] **DA-004** — Supprimer `ZZZ_20260704_tmp_telegram_chatid` (capteur chat_id, désactivé le 04.07, devenu inutile — chat_id trouvé via @rawdatabot)
- [ ] **DA-006** — Supprimer `ZZZ_20260705_tmp_skill_write` (`H79GLAFp0jrx9yrz`, appelant ponctuel de LUMINA-MEMORY-WRITE pour l'écriture du skill n° 990, jamais publié, devenu inutile)
- [ ] **DA-007** — Supprimer `ZZZ_20260705_tmp_budget_test` (`BkQlAcRRRmJQp0tU`, couteau suisse de test de la session du 05.07 au soir : création/neutralisation de fiches ZZTEST, appels de sous-workflows, requêtes Notion ponctuelles — jamais publié, devenu inutile)
- [x] ~~**DA-008**~~ — **tranchée le 05.07.2026** : périmètre v1 = proposition complète (calendrier + copies IG + newsletter + Telegram) **construit et publié** ; newsletter draft → **fiche Notion** (Content & Social) ; Meta Ads + Wix → **Karter fournit les accès maintenant** pour intégration v1.1 (branches en attente des credentials n8n Facebook Graph API + Wix Header Auth)
- [ ] **DA-009** — Supprimer `ZZZ_20260705_tmp_w4_probe` (`Yk8t5R6WQxbr6tmB`, sonde jetable de la session W-4 : lecture de schémas Notion, création/neutralisation de l'édition test, ajout du champ Publish date — jamais publiée, devenue inutile). Nettoyer aussi la fiche `ZZTEST-Marketing-Planner` (Editions, `Cancelled`) et les **12 pages de test dans Content & Social** → corbeille.
- ~~DA-005~~ — **annulée le 05.07 (décision Karter : garder le lookup)** → le workflow est devenu l'utilitaire permanent **`AFTRSN-UTIL-Partners-Sent-Lookup`** (`ZsFiSf7pc5RQAKyc`, dossier `AFTRSN-01-INFRASTRUCTURE`, requête par défaut `in:sent`, manuel, non publié) : recherche dans les envoyés du compte Partners (historique de contact, récupération de fils pour backfill). **Point Hermès : réglé le 05.07 (décision Karter : « Hermès est l'exécutant et l'apprenant — on lui donne le skill directement »)** — skill **n° 990 « Recherche d'historique email envoyé (Gmail) »** écrit dans `lumina_memory` collection `skills` via LUMINA-MEMORY-WRITE (dédoublonnage avant outreach, récupération de fil/date pour relance et backfill, pièges de graphie Gmail, règle lecture seule, section amélioration continue). Vérifié : il sort en tête de la recherche sémantique d'Hermès sur une requête d'usage réel. L'utilitaire `AFTRSN-UTIL-Partners-Sent-Lookup` est référencé dedans comme outil d'exécution.

---

## Rappels transverses (invariants Lumina OS)

- **Numérotation (règle Karter, permanente) : toute numérotation — étapes, chapitres, sections, processus, listes — commence toujours à 1, jamais à 0.**
- PII jamais dans le cloud (`task_type='private'` bloqué par le Router)
- Actions critiques (dépense, envoi email, publication, suppression) toujours gatées par validation humaine
- Embedding figé (`text-embedding-3-small`, 1536 dims) — ne jamais changer
- Git = source de vérité ; pgvector et Notion sont des dérivés régénérables
- n8n fait foi en cas d'écart avec la doc
- Aucune suppression/DROP sans Karter
- Jamais réactiver un `ZZZ_` sans revue complète
- Toujours HTTP Request pour appeler Claude (jamais le node Anthropic natif, bug 404 connu)

---
*Fiche vivante — à mettre à jour à chaque session significative sur le projet AFTRSN Automation.*
