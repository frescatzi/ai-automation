---
type: raw
title: "Spec_AFTRSN-Knowledge-Capture_W7_20260705"
source_url: "drive:1Ah8aZ8jhRz1XC4FuNJ5bLB1z-LFOfc2-"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Spec — AFTRSN-Knowledge-Capture (W-7)

**Statut :** 📐 **SPEC — à valider par Karter avant construction.** Dernier morceau de la plateforme (Étape 4 / P5 Post-event).
**Rédigée :** 05.07.2026 (nuit), après cartographie infra (schéma Editions, contrats mémoire depuis le Maestro).
**Décisions Karter (05.07) :** déclenchement **auto sur passage en `Done`** · débrief en **fiche Notion dédiée** · **agent qui pré-remplit depuis les données** · **spec d'abord**.

---

## 1 · Objectif (P5)

Capturer la connaissance post-event de chaque édition AFTER SUN dans la mémoire longue `aftrsn_memory`, collection **`insights`** — pour que les éditions futures apprennent des précédentes (lieu, lineup, promo, budget, timing, opérations). Le closing budgétaire est déjà couvert par W-6 ; W-7 se concentre sur la **capture de leçons**.

**Invariants respectés :** rien n'est capturé sans validation humaine (gate = passage de la fiche débrief en `Prêt`) · embedding figé `text-embedding-3-small` 1536 dims (jamais changé) · draft/human-gated · mémoire via `LUMINA-MEMORY-WRITE` en Execute Workflow (jamais HTTP direct vers pgvector) · appels LLM via workflow AI-Router (jamais node Anthropic natif).

---

## 2 · Architecture — modèle deux temps (validé Karter)

Un seul workflow planifié (poll quotidien, Europe/Zurich), **deux branches indépendantes** sur le même Schedule Trigger (pattern « chaîne parallèle additive » déjà éprouvé sur le brief J-7).

### Phase A — « Ouvrir le débrief » (auto sur `Done`)
Quand une édition passe en `Status = Done` et n'a **pas encore** de fiche débrief :
1. l'agent (AI-Router) **pré-remplit** une fiche débrief à partir des données Notion déjà connues (date, ville, lieu, capacité cible, lineup, volume de contenu promo, lignes budget) ;
2. la fiche est créée dans la base dédiée **Post-Event Débriefs**, reliée à l'édition, en statut **`À remplir`**, avec les zones qualitatives laissées à compléter par Karter ;
3. ping Telegram : « 📓 Débrief de {Edition} prêt à remplir → {lien} ».

### Phase B — « Capturer » (sur fiche `Prêt`)
Quand Karter a complété la fiche et l'a passée en **`Prêt`** :
1. l'agent (AI-Router, `task_type: extract`) synthétise des **insights structurés** (JSON) depuis le contenu de la fiche + le contexte édition ;
2. chaque insight est écrit dans `aftrsn_memory` collection **`insights`** via `LUMINA-MEMORY-WRITE` ;
3. la fiche passe en **`Capturé`** + `Capturé le` = aujourd'hui (idempotence) ;
4. Telegram : « 🧠 Insights de {Edition} capturés (N) » ; épisode mémoire `episodic` du run.

**Gate humain :** aucune capture tant que Karter n'a pas relu la fiche et posé `Prêt`. C'est la validation critique exigée par l'invariant Lumina OS.

---

## 3 · Base Notion dédiée à créer — « Post-Event Débriefs »

Nouvelle DB dans **⚙️ Operations › 🎭 The Backstage** (même parent que Editions).

| Propriété | Type | Rôle |
|---|---|---|
| `Débrief` | title | « Débrief — {Edition} » |
| `Edition` | relation → Editions (`collection://61418ed4-fb84-48fc-b372-247fbabe0c33`) | lien 1-1 vers l'édition ; sert à l'idempotence Phase A |
| `Status` | select : `À remplir` (yellow) · `Prêt` (blue) · `Capturé` (green) | pilote Phase B + idempotence |
| `Capturé le` | date | horodatage de capture (double garde d'idempotence Phase B) |
| `Note globale` | select : `1`–`5` (optionnel) | ressenti global rapide, indexé plus tard |

**Corps de la fiche (canevas pré-rempli par l'agent en Phase A) :**

```
## Données auto (pré-remplies — ne pas réécrire)
- Édition : {Edition} — {Date} — {City}
- Lieu : {Venue}
- Capacité cible : {Capacity target}
- Lineup : {noms des artistes du Lineup}
- Promo : {n} contenus dans Content & Social ({répartition Post/Story/Reel/Teaser})
- Budget (résumé) : coût {…} / revenu {…} / P&L {…} — détail dans le fichier Sheets de l'édition

## À compléter par Karter (qualitatif)
### Ce qui a marché
-
### Ce qui a raté / frictions
-
### Chiffres réels (fréquentation, entrées, bar…)
-
### Lieu — à refaire / à éviter
-
### Lineup — à refaire / à éviter
-
### Promo — ce qui a converti
-
### Idées pour la prochaine édition
-
```

> Quand la fiche est complétée, Karter la passe en `Prêt` → Phase B la capture au prochain run quotidien.

---

## 4 · IDs & ressources (canoniques — toujours By ID, doublons de bases)

| Ressource | ID |
|---|---|
| Base **Editions** (lecture, filtre `Status=Done`) | `dad7f826-891f-4c29-80e4-183c59bac8b0` (data source `61418ed4-fb84-48fc-b372-247fbabe0c33`) |
| Base **Post-Event Débriefs** (à créer) | `<à renseigner après création>` |
| Base **Content & Social** (comptage promo) | `709b816f-e2f8-4dbf-90ee-597803a11158` (ds `e9960adf-a0c2-4947-947f-23fa65fe9cfe`) |
| Data source **Budget & Finance** | `c8301fab-3b87-4fda-85cf-ab95c2263873` |
| Data source **Lineup / DJs-Artists** | `cc49a4d8-eb50-4f7d-9b6a-37b4623fd87c` |
| Data source **Venue** | `f6239d3d-9c68-4467-95d4-5dd8af8e0437` |
| Credential Notion | `xsBRVxaZ6hhVJWIx` (header `Notion-Version: 2022-06-28`) |
| **LUMINA-MEMORY-WRITE** (Execute Workflow) | `zu4jfZbmDz8trQLl` |
| **LUMINA-AI-Router** (synthèse/pré-remplissage) | `Cu8SozYmondKM8RB` (input `task_type`, `instruction`, `context_raw`) |
| AFTRSN-Culture-Steward (option pressure-test brand) | `vK6B2WuDfbse41Jv` (input `query`) — **optionnel v1.1** |
| Telegram | cred `5kTYhDVm2UDpSDIE`, chat `776345147` |
| Error workflow (à réutiliser) | `xzaH0uWy0idKVphF` |

**Contrat mémoire (Phase B) :**
```
brand: "aftrsn"
collection: "insights"
knowledge_type: "insight"
source: "knowledge-capture"
source_ref: "{Edition}"
title: "{titre court de l'insight}"
content: "{insight complet + contexte édition}"
```
(Modèle calqué sur `Save Episode`/`Save Lesson` du Maestro — mêmes 7 champs, seule la collection change.)

---

## 5 · Détail node-par-node

**Trigger — `Schedule` (Cron `0 9 * * *`, `settings.timezone = Europe/Zurich`).**
Deux branches partent du trigger.

### Branche A — Ouvrir le débrief

| # | Node | Type | Détail |
|---|---|---|---|
| A1 | `Éditions Done` | HTTP Request (Notion) | `POST /v1/databases/dad7f826…/query`, body JSON littéral : `{"filter":{"property":"Status","select":{"equals":"Done"}}}`. **AOD activé** (0 résultat = branche muette propre). |
| A2 | `Débriefs existants` | HTTP Request (Notion) | `POST /v1/databases/<débriefs>…/query` (tous). Sert à construire l'ensemble des `edition_page_id` déjà débriefés. |
| A3 | `Filtrer sans débrief` | Code | Pour chaque édition Done, exclure celles déjà présentes dans A2 (comparaison sur l'id de page d'édition dans la relation `Edition`). Sortie : éditions à ouvrir (0..N). |
| A4 | `Rassembler données` | Code / HTTP | Depuis chaque édition : `Date`, `City`, `Venue` (nom), `Capacity target`, noms du `Lineup`, comptage `Content & Social` par type. *(Résolution des valeurs budget = enrichissement v1.1 si le temps manque : sinon renvoyer le lien du fichier Sheets.)* |
| A5 | `Pré-remplir (AI-Router)` | Execute Workflow → `Cu8SozYmondKM8RB` | `task_type:"draft"`, `instruction:"Rédige le corps d'une fiche débrief post-event AFTER SUN à partir des données fournies : section 'Données auto' factuelle, puis section 'À compléter par Karter' avec titres vides. Ton éditorial sobre, FR."`, `context_raw:` données A4. |
| A6 | `Créer fiche débrief` | HTTP Request (Notion) | `POST /v1/pages` dans la base Débriefs : `Débrief`="Débrief — {Edition}", relation `Edition`, `Status`=`À remplir`, `children`=blocs issus de A5. |
| A7 | `Telegram - Ouvert` | Telegram | « 📓 Débrief de {Edition} prêt à remplir → {url} ». |
| A8 | `Résumé A` | Set (executeOnce) | `{ phase:"open", editions_opened:N }`. |

### Branche B — Capturer

| # | Node | Type | Détail |
|---|---|---|---|
| B1 | `Débriefs Prêt` | HTTP Request (Notion) | `POST /v1/databases/<débriefs>…/query`, filtre `{"property":"Status","select":{"equals":"Prêt"}}`. **AOD activé**. |
| B2 | `Lire fiche` | HTTP Request (Notion) | `GET /v1/blocks/{page_id}/children` → texte du corps de la fiche (réponses de Karter). |
| B3 | `Synthèse insights` | Execute Workflow → `Cu8SozYmondKM8RB` | `task_type:"extract"`, `instruction:"Extrais 3–7 insights généralisables et réutilisables pour de futures éditions (lieu, lineup, promo, budget, timing, ops). Réponds en JSON strict : [{\"title\":\"…\",\"content\":\"…\",\"tag\":\"venue|lineup|promo|budget|ops|timing\"}]. Chaque insight autonome, sans référence à l'incident brut."`, `context_raw:` corps fiche + méta édition. |
| B4 | `Parser JSON` | Code | Nettoyage fences + isolation `[`…`]` ; erreur explicite si non parsable (pattern W-4). Sortie : 1 item par insight. |
| B5 | `Écrire insight` | Execute Workflow → `zu4jfZbmDz8trQLl` (mode `each`) | contrat §4, `collection:"insights"`, `source_ref:{Edition}`. Un appel par insight. |
| B6 | `Marquer Capturé` | HTTP Request (Notion) | `PATCH /v1/pages/{page_id}` : `Status`=`Capturé`, `Capturé le`=today. **Idempotence** : un Capturé n'est jamais re-traité (B1 ne prend que `Prêt`). |
| B7 | `Telegram - Capturé` | Telegram | « 🧠 Insights de {Edition} capturés (N) ». |
| B8 | `Mémoire run` | Execute Workflow → `zu4jfZbmDz8trQLl` | `collection:"episodic"`, résumé du run (éditions ouvertes + capturées). |
| B9 | `Résumé B` | Set (executeOnce) | `{ phase:"capture", debriefs_captured:M, insights_written:K }`. |

**Branches d'erreur :** `onError=continue` sur A5/A6/B3/B5/B6 (un échec LLM/Notion ne casse pas la chaîne, ne laisse pas d'écriture partielle silencieuse) ; `errorWorkflow = xzaH0uWy0idKVphF`.

---

## 6 · Idempotence (double garde)

- **Phase A** : une édition Done qui a déjà une fiche débrief reliée → ignorée (A3). Re-run quotidien = 0 création tant que la fiche existe.
- **Phase B** : seules les fiches `Prêt` sont capturées ; à la capture, passage en `Capturé` + `Capturé le`. Un re-run = 0 écriture mémoire.

Conséquence : le workflow peut tourner tous les jours sans risque de doublon, et rejoue proprement si Karter remet une fiche en `Prêt` après correction (retour volontaire).

---

## 7 · Synergie avec W-6 (Budget-Tracker)

W-6 fait déjà la **clôture auto J+3** (passage `Status=Done`). C'est précisément ce passage que la Phase A détecte : W-6 clôt → W-7 ouvre le débrief le lendemain matin. Aucun couplage direct entre les deux workflows (chacun poll Notion), juste un enchaînement naturel par l'état de l'édition. Pas de collision : W-6 lit/écrit Budget & Finance, W-7 lit Editions + écrit la base Débriefs.

---

## 8 · Tests prévus (avant publication)

1. **Chemin nominal A** : une édition test en `Done` sans débrief → fiche créée en `À remplir`, corps pré-rempli, Telegram reçu.
2. **Idempotence A** : 2e run → 0 création.
3. **Chemin nominal B** : fiche passée en `Prêt` (corps rempli) → N insights écrits dans `insights`, fiche en `Capturé`, Telegram reçu ; vérifier par recherche sémantique mémoire que les insights ressortent.
4. **Idempotence B** : 2e run → 0 écriture (fiche `Capturé` ignorée).
5. **Édition Done sans rien à dire** : fiche `Prêt` quasi vide → l'agent renvoie 0 insight → pas d'écriture, fiche marquée `Capturé` quand même (évite le retraitement).
6. **Vérification par les effets** (relire Notion/Telegram/mémoire, pas les traces d'exécution).

Nettoyage post-test : fiches `ZZTEST-*` neutralisées en `Capturé` (statut ignoré) puis corbeille par Karter ; insights de test à retirer de `insights` si écrits.

---

## 9 · Décisions ouvertes / à confirmer au build

- **DA-W7-a** — Nom exact de la base : « Post-Event Débriefs » proposé (ou « Débriefs » / « Retours d'édition »). À trancher.
- **DA-W7-b** — Résolution des **valeurs budget** dans le pré-remplissage : v1 (lien Sheets seulement) ou v1.1 (chiffres résolus depuis Budget & Finance) ?
- **DA-W7-c** — `Note globale` 1–5 : la garder ou non.
- **DA-W7-d** — Endpoint Notion `databases/{id}/query` vs API data-source 2025 : à vérifier live (W-4 a utilisé l'id base `dad7f826…` avec succès → on part là-dessus, vérif au build).
- **DA-W7-e** — Culture-Steward en pressure-test brand des insights avant écriture : v1.1 optionnel.

---

## 10 · Definition of Done (rappel marche à suivre §5)

Tous les nodes présents/connectés/renommés · aucun credential saisi par Claude · épisode mémoire `episodic` en fin de run · branche d'erreur sans écriture partielle · testé end-to-end les deux phases · publié dans n8n · Fiche technique + POS paire produits · Fiche Projet + passation mises à jour.

---
*Spec rédigée le 05.07.2026 (nuit). À valider par Karter, puis construire dans n8n (navigateur) selon `AFTRSN_MarcheASuivre_Construction-n8n.md`.*
