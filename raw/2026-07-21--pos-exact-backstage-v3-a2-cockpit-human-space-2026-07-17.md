---
type: raw
title: "POS-EXACT_Backstage-v3_A2-cockpit-human-space_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_a2-cockpit-human-space_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · Amendement 2 « Cockpit dans Human Space » (accueil portail pur, sidebar propre)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **FAIT & VALIDÉ par Karter le 17.07.2026** (décision : « je privilégie la clarté »)
Référence : `SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md` · `CONCEPT_Backstage-v3_Deux-Mondes_2026-07-16.md` (§0 re-amendé : le cockpit REVIENT dans Human Space)

## Contexte

Le 16.07 au soir (crédits épuisés), Karter a fait des modifications **manuelles** : recréation d'une sous-page Cockpit dans Human Space + tentatives de renommage qui ont déclenché le piège « renommer un wrapper renomme la base partagée » sur **3 data sources** (Tasks → « ⏭️ Up Next », Editions → « 🎪 Upcoming Editions », Content & Social → « 📣 Publishing Pipeline »). Demande initiale de la session : cockpit affiché sur l'accueil ET sidebar réduite à Backstage/Human Space/Agents Space/Archive. Le test synced block a prouvé que ces deux exigences sont **structurellement incompatibles** dans Notion → décision Karter : la clarté gagne, le cockpit vit dans Human Space, l'accueil redevient un portail pur.

## État final atteint

- **The Backstage (accueil)** = callout identité → 🗂️ Zones (2 portes : 👩🏽‍💼 Human Space · 🤖 Agents Space) → 🗄️ Archive. Rien d'autre.
- **Human Space** = 🚪 Rooms : **🔭 Cockpit (`3a024f2f-ef0c-8118-80ab-c44d93238ca6`)** / Operations / Knowledge / Team.
- **🔭 Cockpit** = intro + base **Tasks** (complète, tous onglets) + ⏭️ Up Next + 🎪 Upcoming Editions + 📣 Publishing Pipeline + `## ✅ Awaiting validation` + vue To Validate.
- **Sidebar** = The Backstage → Human Space · Agents Space · Archive (les 5 vues nichées sous Human Space → Cockpit).
- **3 data sources réparées** : `e50f931d…` = « Tasks » · `61418ed4…` = « Editions » · `e9960adf…` = « Content & Social ». Relations/rollups/vues intacts (vérifié).

## Marche à suivre exacte

1. **Réautorisation du connecteur Notion** par Karter (jeton expiré depuis le 16.07 ~00:20).
2. **Lire l'état réel avant d'agir** : `notion-fetch` de l'accueil `39124f2fef0c8172a961c8a57721ff0f`, de Human Space `39124f2fef0c811599edec1c18b44b20` et du Cockpit manuel — l'état avait divergé de la mémoire (modifs manuelles de Karter). Ne jamais présumer que l'état documenté = l'état réel.
3. **Réparer les noms des 3 data sources** : `notion-update-data-source` avec `title` : `e50f931d-3f8d-483f-895a-c09408455128` → « Tasks » · `61418ed4-fb84-48fc-b372-247fbabe0c33` → « Editions » · `e9960adf-a0c2-4947-947f-23fa65fe9cfe` → « Content & Social ». Vérifier le schéma retourné (relations/rollups intacts).
4. **Créer 🔭 Cockpit** : `notion-create-pages` parent Human Space → `3a024f2f-ef0c-8118-80ab-c44d93238ca6` (icône 🔭, intro italique).
5. **Déménager les 5 blocs** dans Cockpit : `notion-move-pages` sur `cd210f519f9a4d67a50ef6f50e851dc5` (base Tasks) + wrappers `39e24f2fef0c817ab3b5f50fee65eb1f` (Up Next) · `39e24f2fef0c81d9b9bde18dff28b75c` (Upcoming Editions) · `39e24f2fef0c81519288e8db47d61f53` (Publishing Pipeline) · `39f24f2fef0c81209f84e79ca8c809b3` (Awaiting validation).
6. **Structurer le Cockpit** : `replace_content` — intro + les 5 blocs relistés + titre `## ✅ Awaiting validation` + description au-dessus de la vue To Validate.
7. **Accueil portail pur** : `replace_content` — callout → `## 🗂️ Zones` + description + colonnes (2 portes) → divider → page Archive. (Relister les 3 pages enfants, sinon suppression.)
8. **Human Space** : `replace_content` — intro + `## 🚪 Rooms` : Cockpit / Operations / Knowledge / Team.
9. **Fix cosmétique navigateur** : le header de la base Tasks affichait encore « ⏭️ Up Next » (override de bloc posé lors du renommage manuel, distinct du nom de la data source) → double-clic sur le header → `cmd+a` → « Tasks » → Escape.
10. **Vérifications** : reload → accueil (portail pur) ✅ · sidebar (3 items sous Backstage) ✅ · Cockpit (5 vues peuplées : board Tasks avec pastilles Executor, EP6 dans Upcoming Editions, Awaiting validation avec la tâche To Validate ; Publishing Pipeline vide = légitime, filtre Approved/Scheduled/Brand-check sans résultats — config vérifiée par API) ✅ · pages test corbeillées ✅.

## Difficultés rencontrées

- **Exigences contradictoires découvertes en cours de route** : « cockpit affiché sur l'accueil » ET « sidebar propre sous Backstage » — impossible ensemble (voir lessons learned). Deux allers-retours (blocs remontés sur l'accueil puis re-descendus) avant le test décisif.
- **Dégâts silencieux des modifs manuelles** : 3 data sources renommées sans que Karter le sache ; doublon « Up Next » dans la sidebar (= la base Tasks renommée + le wrapper).
- Le déballage d'un synced block par `replace_content` est bloqué par le garde-fou (il croit qu'on supprime les 5 bases).
- Le renommage API de la data source ne corrigeait pas le header affiché (override de bloc distinct).
- Scroll Notion erratique + état « page still loading » bloquant les outils à injection de script (scroll/find) — les clics et captures passaient.

## Solutions implémentées

- **Test empirique avant décision** (page brouillon sous Archive) : synced block + linked view dedans (navigateur), synced reference sur l'accueil → rendu OK mais sidebar polluée → preuve de l'incompatibilité → choix éclairé de Karter.
- **`notion-move-pages` vers le MÊME parent** pour sortir des blocs d'un synced block (ils re-deviennent enfants directs de la page) — le déballage que `replace_content` refuse.
- Réparation en 2 couches du nom Tasks : data source par API + override de header au navigateur.
- Contournement du scroll bloqué : `cmd+Down` pour la page, repli des sections sidebar (Meetings/Agents) pour révéler le bas de la liste, `zoom` pour inspecter sans scroller.

## Lessons learned

- **LOI SIDEBAR NOTION : toute base/vue VISIBLE sur une page est listée dans la sidebar sous cette page** — y compris via synced block reference, y compris repliée dans un toggle. « Contenu affiché sur l'accueil » et « sidebar minimale » sont donc mutuellement exclusifs. Le seul curseur : où l'on met le contenu.
- **Une linked view PEUT vivre dans un synced block** (création navigateur : /synced puis /linked dedans ; l'API ne sait pas créer de vue à l'intérieur). La reference rend les vues pleinement interactives ailleurs — pattern valable quand la pollution de sidebar est acceptable.
- **`replace_content` sait ENVELOPPER des blocs existants dans un `<synced_block>`** (API) mais refuse de les déballer → `move-pages` vers le même parent fait le déballage proprement.
- **Le header d'une base inline a un nom d'override de bloc**, indépendant du nom de la data source : après un renommage accidentel, réparer LES DEUX (API pour la source, navigateur pour le header).
- **Après toute session manuelle de l'utilisateur, re-fetch l'état réel avant d'agir** — la mémoire/spec décrit l'état d'avant, pas l'état présent. Ici : 3 data sources renommées + structure recréée découvertes uniquement par lecture.
- Reformuler la demande + tester la faisabilité AVANT d'exécuter (demandé explicitement par Karter) évite les allers-retours destructeurs — les deux moves inutiles de cette session datent d'avant cette discipline.
