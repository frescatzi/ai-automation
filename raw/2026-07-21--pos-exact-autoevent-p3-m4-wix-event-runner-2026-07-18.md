---
type: raw
title: "POS-EXACT_AutoEvent-P3_M4-wix-event-runner_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m4-wix-event-runner_2026-07-18.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Automatisation Événementielle P3 · M4 — Wix Event Runner (page événement Wix en draft)

**Date :** 18.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Milestone :** M4 · **Statut :** ✅ TESTÉ, VALIDÉ & ACTIF

---

## 1. Objectif (tel que tranché avec Karter)

Produire par n8n une **page événement Wix en DRAFT** pour une édition, **jamais publiée** (la publication reste un geste humain derrière le gate). La page reprend la copy produite par M2/Iris (`Wix short description` + `Wix long description`) et les infos d'édition (titre, dates/heures, ville, type). Deux chantiers :

- **Chantier 1 — création** : à la première copy « Announce » d'une édition, créer l'event Wix en draft (RSVP si l'édition est **Free**, lien billetterie **EXTERNAL** si **Paid**), puis écrire en retour l'`Wix event ID` + l'`Wix draft URL` sur l'édition (link-back).
- **Chantier 2 — mise à jour** : à chaque nouvelle copy (ex. « Line-up reveal »), **mettre à jour** la page Wix existante (description longue/courte) au lieu d'en recréer une.

Déclenchement **par la copy** (base Content & Social), pas par la tâche : dès qu'Iris pose un pack de copy, la page Wix suit automatiquement.

## 2. Ce qui a été construit

### 2.1 Schéma Notion (additif, zéro casse)
- **Editions** (ds `61418ed4-fb84-48fc-b372-247fbabe0c33`) : ajout **`Event-type`** (select **Free / Paid**), **`Ticket URL`** (url, lien billetterie externe si Paid), **`Wix event ID`** (text, écrit par l'automate = link-back + clé d'idempotence create/update), **`Wix draft URL`** (url, écrit par l'automate = accès direct au draft). Réutilisés : `Format` (Daytime/Day-to-Night/Nightclub), `Date` (config native Notion **date + heure, début → fin**, ex. 18/07 20:00 → 19/07 04:00), `City`, `Venue`.
- **Content & Social** (ds `e9960adf-a0c2-4947-947f-23fa65fe9cfe`) : source des champs `Wix short description` + `Wix long description` (produits par M2). Aucun ajout.
- **Garde-fou = `Status` de l'édition** (décision Karter) : à partir de **Live / Cancelled / Done**, aucune modification (voir M1 pour le gel). Planning / Confirmed = actif.

### 2.2 Credential + API Wix
- **Credential n8n** `WIX AFTRSN-LUMINA` = `2SjAjkAPXNNjUQYL`, type **HTTP Custom Auth** (Generic). En-têtes : `Authorization: <clé API compte Wix>` + `wix-site-id: c895f7a6-9708-4fe4-a85b-55add4f393eb`. **Secret collé par Karter uniquement** (jamais en clair dans le chat ni les docs). Events est **au niveau site** → pas besoin de `wix-account-id`.
- **API** : Wix **Events V3 REST**. Create `POST https://www.wixapis.com/events/v3/events` avec `draft:true`. Update `PATCH https://www.wixapis.com/events/v3/events/{id}` (partiel, **sans** champ revision). Get `GET …/{id}?fields=TEXTS` (⚠️ `fields=TEXTS` sinon les descriptions sont masquées). Registration : **`registration.initialType`** (et NON `type`) = `RSVP` (`rsvp.responseType:'YES_ONLY'`) pour Free, `EXTERNAL` (`external.url`) pour Paid. Fuseau `Europe/Zurich`.

### 2.3 Workflow n8n
- **`AFTRSN-WIX-EVENT-RUNNER`** = `6XVotFtRAZ7BmLBk` — **ACTIF**, Error Workflow = Sentinel `xzaH0uWy0idKVphF`, 19 nœuds. Credential Notion `sCsj206WVkNxO0Jl` + credential Wix `2SjAjkAPXNNjUQYL`.
- **Déclenchement** : 2 Notion Triggers (Added + Updated) sur **Content & Social** DB `709b816fe2f84dbf90ee597803a11158`, poll **1 min**. (Chemin fiable = Added : chaque pack de copy est une page neuve.)
- **Chaîne** : Read the copy (get) → **Should build?** (code, itère `$input.all()` : extrait `wixShort`/`wixLong` par regex de clé, `editionId` = 32 derniers hex de la relation Edition ; skip si pas de copy ou pas d'édition) → Read the edition → **Prepare event** (code : détermine la **route** — `update` si `Wix event ID` présent ; `blocked` si `Event-type` vide, ou Paid sans `Ticket URL`, ou dates début/fin manquantes ; sinon `create` ; construit `taskQueryBody` pour retrouver la tâche « Wix event page » To Do de l'édition) → **Find Wix task** (HTTP POST query Notion Tasks) → **Attach task** (récupère `taskPageId`) → **Is update?** (IF route==update) :
  - **branche update** → Build update (body `{event:{id, shortDescription?, detailedDescription?}}`) → **Update Wix event** (PATCH) → **Mark task To Validate (upd)**.
  - **sinon Buildable?** (IF route==create) → Build event (body `{event:{title, dateAndTimeSettings, registration, shortDescription?, detailedDescription?, location}, draft:true}`) → **Create Wix event (draft)** (POST) → **Parse create** (extrait `id` + URL page ; construit le patch Notion) → **Write Wix refs to edition** (HTTP PATCH page Notion → `Wix event ID` + `Wix draft URL`) → **Mark task To Validate**.
  - **sinon** → **Mark task Blocked** (Status Blocked + Notes = raison).
- **Idempotence create/update** : la présence de `Wix event ID` sur l'édition route en `update` → jamais de doublon de page ; les updates sont partiels.

## 3. Refactor FIABILITÉ (fait dans la même session, à la demande de Karter)

Contexte : le trigger Notion **« Updated » ne mord pas de façon fiable** (dé-duplication par page dans `staticData.possibleDuplicates` → une page déjà vue n'est pas re-déclenchée). Karter : « si le trigger Updated ne mord pas, on ne pousse rien, on fixe la racine ». Deux corrections de fond :

1. **M2 (Copy Runner) — line-up non transmis à Iris.** `Build the query` lisait **uniquement** le champ texte `Lineup notes` (vide) et **jamais la relation `Lineup`** (où sont réellement les DJs). Sur une vague « Line-up reveal », Iris recevait « Line-up: not announced yet » + la consigne « nomme les DJs locaux » → contradiction → Iris **refuse d'inventer** (règle stricte) → tâche `Blocked` → pas de copy → M4 jamais déclenché → page Wix figée. **Fix** : ajout d'un nœud **`Read line-up roster`** (getAll DJs/Artists DB `f27a9819d49f489dbf25277e3381d564`) et résolution des IDs de la relation → **noms** (Mila, SAM B) injectés dans le brief pour les vagues reveal/update.
2. **M2 — dépendance aux triggers Notion Added/Updated.** Basculé sur le **même modèle que M1** : **Schedule Trigger toutes les 1 min → `List tasks` (getAll) → `Should draft?`** (itère toutes les tâches, filtre `Copy & captions` + `To Do` + `Agent` + event). Plus aucune dépendance à « Updated » ; toute tâche éligible (créée, modifiée, **ou relancée**) est reprise au prochain scan. `Should draft?` sort N items, l'aval traite le premier éligible par run → **une tâche par minute**, sans doublon (la tâche quitte `To Do` en fin de run).

## 4. Tests réalisés

- **Vague 1 (create) — AUTO de bout en bout** sur édition **EP07 - Event TEST-AUTO** (Planning, Free, City Zürich, Venue Soluna, Date+heures début→fin) : M1 pose 5 tâches Announce → M2/Iris écrit le pack de copy → M4 crée l'event Wix draft `a3a02669-e6b0-46a8-b061-b4ad0047fb44` (URL `https://www.aftersunpeople.com/event-details/ep07-event-test-auto`), `Wix event ID` + `Wix draft URL` écrits sur l'édition, tâche « Announce · Wix event page » → **To Validate**, statut Wix **DRAFT**.
- **Vague 2 (update)** : ajout du line-up (relation Lineup = **Mila** + **SAM B**) → M1 pose la vague « Line-up reveal ». **Échec initial révélateur** : la tâche « Line-up reveal · Copy & captions » est passée **Blocked** (Iris a refusé faute de noms) → confirmation du bug §3.1. Après fix §3.1 + bascule scan §3.2, la tâche relancée a été reprise par le scan M2 → Iris a réécrit la copy avec **SAM B puis Mila** (DJs locaux d'abord, sans hype, voix de marque) → M4 a **mis à jour** la page Wix (PATCH, `updatedDate` 22:29, toujours **DRAFT**), tâche « Line-up reveal · Wix event page » → **To Validate**. Vérifié par `GET …?fields=TEXTS` : `detailedDescription` nomme bien Mila et SAM B.

## 5. Difficultés rencontrées
1. **Wix registration** : `POST` en 400 « registration is invalid: initialType value is required » — on avait utilisé `type` au lieu de **`initialType`**.
2. **Credential Wix** : « Invalid Custom Auth JSON » (guillemets manquants) puis 428 `MISSING_REQUEST_SITE_CONTEXT` (clé API tronquée d'une lettre) — causes de saisie du secret, pas de permissions.
3. **GET masque les descriptions** : `?fields=REGISTRATION` renvoyait sans les textes ; il faut **`fields=TEXTS`**.
4. **Date sans heure** : une édition n'avait qu'une date ; les events ont toujours des heures (variables selon format/ville/lieu) → il fallait **date + heure, début → fin**.
5. **Trigger « Updated » qui ne mord pas** : re-déclenchement non garanti (dé-dup par page) → une relance de tâche (Blocked → To Do) n'était jamais reprise. C'est le défaut structurel signalé par Karter.
6. **Line-up dans la relation, pas dans le texte** : M2 ne lisait pas la relation → Iris privé des noms → blocage en cascade jusqu'à la page Wix figée.
7. **`shortDescription` non persistée côté Wix** : bien qu'envoyée (create et update), le `GET fields=TEXTS` renvoie `shortDescription:""` (la description longue, elle, passe). Comportement Wix V3 à confirmer.

## 6. Solutions implémentées
1. `registration.initialType` = `RSVP` (Free) / `EXTERNAL` (Paid) ; `draft:true` systématique.
2. Secret Wix re-saisi proprement par Karter (Authorization + wix-site-id) → HTTP 200.
3. Lectures Wix en `fields=TEXTS`.
4. Champ `Date` natif Notion en **plage datetime** (début→fin) ; `Prepare event` bloque si dates manquantes.
5. **M2 basculé en scan planifié** (patron M1) → indépendant du trigger « Updated » ; toute relance est reprise. (M4 reste sur Added, fiable car chaque copy est une page neuve ; bascule scan de M4/M3 = suite planifiée.)
6. **`Read line-up roster` + résolution relation→noms** dans le brief M2 (vagues reveal/update).
7. `shortDescription` conservée dans le code (inoffensive) ; **résidu** documenté (patch/format à valider côté Wix).

## 7. Lessons learned
- **Toujours résoudre les relations en valeurs, pas en IDs, quand un agent doit les lire.** Une empreinte d'ID suffit à détecter un changement, pas à rédiger : il faut le **nom**.
- **Ne jamais dépendre du trigger Notion « Updated » pour une reprise.** Il dé-duplique par page ; pour une reprise fiable (création, édition, relance), utiliser un **scan planifié + requête d'état** (patron M1). Idempotence par mutation d'état (la tâche quitte `To Do`).
- **Un blocage propre est un bon signal.** Iris refusant d'inventer a révélé le vrai bug au lieu de produire du faux. Le garde-fou « Blocked + raison » a servi de diagnostic.
- **Le link-back ferme la boucle create↔update** : écrire `Wix event ID` sur l'édition rend l'idempotence triviale (présence = update).
- **Wix Events V3** : `initialType` (pas `type`), site-level (pas d'account-id), PATCH partiel sans revision, lectures en `fields=TEXTS`.
- **Secrets = Karter only**, toujours (aucune clé en clair).

## 8. IDs clés M4
- **n8n** : `AFTRSN-WIX-EVENT-RUNNER` = `6XVotFtRAZ7BmLBk` (actif, 19 nœuds) · Error WF Sentinel `xzaH0uWy0idKVphF` · cred Notion `sCsj206WVkNxO0Jl` · cred Wix Custom Auth `2SjAjkAPXNNjUQYL`.
- **Wix** : site id `c895f7a6-9708-4fe4-a85b-55add4f393eb` · Events V3 (`/events/v3/events`) · event test EP07 `a3a02669-e6b0-46a8-b061-b4ad0047fb44`.
- **Notion** : Editions ds `61418ed4-fb84-48fc-b372-247fbabe0c33` (M4 : `Event-type`, `Ticket URL`, `Wix event ID`, `Wix draft URL`) · Content & Social DB `709b816fe2f84dbf90ee597803a11158` / ds `e9960adf-a0c2-4947-947f-23fa65fe9cfe` · Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` · DJs/Artists DB `f27a9819d49f489dbf25277e3381d564` / ds `cc49a4d8-eb50-4f7d-9b6a-37b4623fd87c`.
- **M2 refactor** : `AFTRSN-COPY-RUNNER` = `XQYvzTzFK2NLjDZz` (scan planifié 1 min + `Read line-up roster` + résolution relation→noms).

## 9. Résidus / suite
- **`shortDescription` Wix vide** : valider le champ/format V3 (ou l'abandonner si non exposé) — la description longue suffit à l'affichage.
- **Fiabiliser M3 + M4** : les basculer sur scan planifié comme M1/M2 (M4 encore sur triggers Content & Social ; fonctionne via Added mais expose au même défaut « Updated »).
- **Nettoyage test** : édition **EP07 - Event TEST-AUTO** + ses tâches/copies + le draft Wix `a3a02669-…` — à supprimer (confirmer avec Karter avant suppression).
- **Anti-doublon scan** : Iris ~20-30 s < 60 s → pas de recouvrement observé ; si un run dépassait 1 min, ajouter un « claim » (tâche → In Progress) en tête de M2. À surveiller.
- **Billetterie Paid** : chemin `EXTERNAL` codé, non encore testé sur une vraie édition payante.
