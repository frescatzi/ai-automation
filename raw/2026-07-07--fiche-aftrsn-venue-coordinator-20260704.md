---
type: raw
title: "Fiche_AFTRSN-Venue-Coordinator_20260704"
source_url: "drive:1K8VC85giW_2QGMx03XYULYMI6d3v7veg"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche workflow — AFTRSN-Venue-Coordinator

**Format :** conforme à la Bible 360° LUMINA OS §E.

---

**ID :** `Su9tO0fS39u9itlU` — **Criticité :** 🔴 (critique — deadline venue/artistes du 18.07)
**Statut :** 🟢 **publié le 04.07.2026** (« v1.0 — Venue-Coordinator », v1.0.1 fuseau Zurich, v1.1 « from day to night », v1.2 déclenchement par statut « To contact », **v1.2.1 espacement relance 2 le 05.07.2026**)
**Emplacement :** `Personal / 02-AFTRSN / AFTRSN-02-BUSINESS AUTOMATION / AFTRSN-PROCESS-03-Venue-Coordinator`
**Tags :** 🌞 AFTRSN · 🔄 automation · 🟢 PROD
**Déclencheur :** Schedule — jours ouvrés à 9h, fuseau **Europe/Zurich** (fixé explicitement, l'instance est en UTC)

## Rôle

Processus transverse « Coordination lieux » : chaque matin ouvré, lit la base **Venues** de 🎭 The Backstage, détermine pour chaque lieu en statut `Prospect` (avec email) l'action due — **premier contact**, **relance à J+3**, ou **seconde relance à J+7** — fait rédiger l'email par **AFTRSN-Secretary** (allemand professionnel + version courte anglaise), dépose un **brouillon** dans le Gmail **Partners** (relances dans le même fil), met à jour la fiche Notion (dates + fil Gmail), notifie Karter sur Telegram, journalise l'épisode. **Aucun envoi automatique, jamais** — Karter relit et envoie depuis Gmail.

## Pilotage par la base Venues (source de vérité)

| Champ Notion | Rôle |
|---|---|
| `Status` | **`Prospect` = réserve silencieuse (rien ne part)** ; **`To contact` = déclencheur posé par Karter** → premier contact + relances ; **tout autre statut (`In discussion`…) stoppe les relances**. Règle Karter (05.07.2026) : le rythme (~2–3 premiers contacts/semaine) est une décision humaine, volontairement **hors process** — c'est Karter qui dose en posant le statut |
| `Email` | adresse d'outreach (champ ajouté le 04.07.2026) — sans email, le lieu est ignoré |
| `1st Email` | posé automatiquement au premier brouillon ; les relances se calculent à partir de cette date |
| `Follow-up 1` / `Follow-up 2` | posés automatiquement à chaque relance (une seule fois chacun) |
| `Gmail Thread` | id du fil Gmail (champ ajouté le 04.07.2026) — permet les relances dans le même fil |
| `Contact` / `Notes` / `Capacity` | contexte injecté dans le prompt Secretary |

**Logique de décision** (node `Actions du jour`, v1.2.1) : seuls les lieux `To contact` avec email sont considérés ; pas de `1st Email` → premier contact ; `1st Email` ≥ 3 jours et pas de `Follow-up 1` → relance 1 ; `Follow-up 1` ≥ 4 jours et pas de `Follow-up 2` → relance 2 ; sinon rien. Idempotence garantie par les dates écrites en fin de chaîne. **Backfill supporté** : un lieu contacté manuellement avant l'automatisation s'intègre en posant `Email`, `1st Email` (date réelle), `Gmail Thread` (fil d'origine) et `To contact` — les relances reprennent dans le fil (fait le 05.07.2026 pour Maison 25, Soluna, The Palm Three).

## Chaîne (17 nodes)

`Jours ouvres a 9h` → `Notion - Lieux en base` (getAll, DB Venues `b9001abd-…`) → `Actions du jour` (Code : filtre Prospect+email, calcul de l'action — ⚠️ sortie Notion simplifiée = champs `property_*` en snake_case, titre dans `name`) → `Secretary - Rediger` (Execute Workflow → `AFTRSN-Secretary`, sortie balisée `SUJET:` / `RESUME:` / `---DRAFT---`) → `Decouper la reponse` (Code, repli sur texte brut si balises absentes) → `Aiguillage` (Switch 3 branches) →
- **Premier contact** : `Gmail - Brouillon nouveau` → `Notion - Noter 1er email` (date + `Gmail Thread` ← threadId du brouillon)
- **Relance 1** : `Gmail - Brouillon relance 1` (options.threadId) → `Notion - Noter relance 1`
- **Relance 2** : `Gmail - Brouillon relance 2` (options.threadId) → `Notion - Noter relance 2`

→ `Rejoindre` (Merge append 3 entrées) → `Telegram - Notifier` (un message par lieu : 🏛️ / 📍 Lieu / 🔁 Type / ✉️ Objet / 📬 Destinataire / 📝 Résumé — zéro code technique, règle Karter) → `Compter` → `Memoire - Save Episode` (`aftrsn_memory`, `episodic`, source_ref `AFTRSN-Venue-Coordinator`) → `Resume` (`{status, drafts}`).

**Comportement à vide :** aucune action due → arrêt silencieux après `Actions du jour` (ni notification ni épisode) — voulu, validé en test.

## Règles du prompt Secretary (v1.1 — 05.07.2026)

Corps en allemand (vouvoiement Sie, professionnel chaleureux) puis `---` puis version courte anglaise ; présentation AFTER SUN PEOPLE comme concept **« from day to night »** (règle métier Karter, 05.07.2026 : la culture du jour n'est favorable que ~2 mois d'été en Europe ; le reste de l'année le format bascule vers la nuit — précédents Londres et Turquie en boîte de nuit, même nom, même direction artistique) ; direction musicale Afro House / Afro Tech / Melodic Afro House ; le format proposé s'adapte au lieu (café-bar/terrasse → jour/soirée ; club → soirée/nuit ; gabarits 15h–22h, 19h–02h, 22h–04h/06h) ; modalités ouvertes : location, co-production ou guest night ; demande de disponibilités et conditions ; personnalisation 1 phrase max depuis les Notes ; relances brèves et courtoises ; **jamais d'engagement ferme, montant, date garantie ou accord contractuel** ; signature AFTER SUN PEOPLE + aftersunpeople.com.

## Dépendances & credentials

- **Notion** : credential `Notion account` (`xsBRVxaZ6hhVJWIx`), DB Venues 🎭 By ID
- **Gmail** : credential `PARTNERS-GMAIL` (`ohE4nLNpeFYMG9Ha`) — compte **Partners** (B2B), créée et connectée par Karter le 04.07.2026 (même client OAuth que Hello, API Gmail déjà active sur le projet `909839129866`)
- **AFTRSN-Secretary** (`FtPFOKvi2t18DFb1`), contrat `{query}` — **LuminaOsBot - Telegram** (`5kTYhDVm2UDpSDIE`), chat `776345147` — **LUMINA-MEMORY-WRITE** (`zu4jfZbmDz8trQLl`)
- Aucun credential saisi par l'agent.

## Construit / testé / publié le 04.07.2026

Construit par l'API REST interne n8n (POST unique, 17 nodes). 4 tests : premier contact ✅ (brouillon + `1st Email` + fil capturé), relance J+3 ✅ (même fil, `Follow-up 1`), relance J+7 ✅ (même fil, `Follow-up 2`), à-vide ✅ (arrêt propre). Données de test : fiche `ZZTEST-Venue-Coordinator` neutralisée (suppression manuelle Karter) + 3 brouillons Gmail à supprimer par Karter. Publié avec fuseau Europe/Zurich explicite.

## ⚠️ Ne pas toucher sans précaution

- Les relances se calculent sur la date du **brouillon**, pas de l'envoi réel. Si Karter envoie un premier contact plusieurs jours après sa création, ajuster `1st Email` dans Notion pour recaler les relances.
- **Stopper les relances = sortir le lieu du statut `To contact`** (`In discussion`, `Confirmed`, retour à `Prospect`…) — c'est le seul mécanisme (pas de détection automatique des réponses en v1). Un lieu laissé en `To contact` reçoit ses 2 relances puis s'arrête de lui-même.
- Si `Gmail Thread` est vidé manuellement, une relance due partirait avec un threadId vide → erreur du node Gmail. Ne pas vider ce champ.
- Le format balisé `SUJET:` / `RESUME:` / `---DRAFT---` est parsé par `Decouper la reponse` — garder les balises si le prompt change.
- La sortie du node Notion getAll est en format simplifié `property_*` snake_case — tout nouveau champ Notion utilisé dans le code doit suivre cette convention.
