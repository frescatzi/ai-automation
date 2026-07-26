---
type: raw
title: "POS-EXACT_AutoEvent-P3_M1-orchestrateur-edition-checklist-brief_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m1-orchestrateur-edition-checklist-brief_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · M1 · Orchestrateur : Edition → checklist + Brief (2 vagues)

**Date :** 17.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Espace :** n8n LUMINA OS + Notion « The Backstage » (AFTER SUN PEOPLE - HQ)
**Statut :** ✅ TERMINÉ & VALIDÉ Karter (17.07.2026). Milestone M1 de la Phase 3 (moteur d'automatisation événementielle).
**Workflow :** `AFTRSN-IRIS-EVENT-AGENT` = `git92S6gDU1hGZ2b` (à l'origine `AFTRSN-EVENT-ORCHESTRATOR` ; renommé après le passage à l'agent Iris, cf. POS M1b).

---

## 1. Objectif

Partir d'une **Edition** Notion (l'événement) et faire générer par n8n, en un clic, le **brief consolidé** + la **checklist de production** des chantiers de com, chaque tâche déposée en `To Do` côté `🤖 Agent`, sans qu'aucun automate n'écrive « Done » ni ne publie quoi que ce soit. Chantier 100 % interne (zéro connecteur externe), le plus sûr, pour prouver la tuyauterie.

## 2. Décisions de cadrage tranchées (section 1 de la spec, 17.07)

- **D1 Newsletter** : Wix natif (brouillon préparé, envoi manuel derrière le gate).
- **D2 Instagram** : programmation + post manuel (compte sensible, zéro config Meta lourde).
- **D3 Billetterie** : selon l'édition. Locales = gratuites/inscription ; internationales = payantes, lien billetterie posé directement sur la page événement Wix.
- **D4 Canva** : plan Pro/Teams → API Connect + Brand Templates jouables (M3).
- **D5 Facebook Ads** : manuel au début (risque budget), on ne branche l'API Meta que plus tard sur OK budget explicite.

## 3. Le modèle « deux vagues » (co-conçu avec Karter)

Le line-up (comme la tagline, l'artwork, le lien billetterie) arrive souvent 1 à 2 semaines APRÈS la création de l'événement. **L'absence de line-up ne bloque rien** : elle définit une PHASE de contenu. Il y a donc deux vagues :

- **Vague 1 · Announce** : date + lieu connus, visuel sans DJ, site publié → contenu d'annonce (save-the-date, page event, teaser flyer sans line-up, post + story). Publiable tout de suite.
- **Vague 2 · Line-up reveal** : line-up confirmé → flyer complété avec les DJ, artwork du site mis à jour, post/story de révélation, newsletter. Un DEUXIÈME temps fort, pas un déblocage. « Au lieu d'un contenu, on en a deux, ça fait vivre la page. »

### Champ de contrôle Notion « Content Phase »
Ajouté sur la base Editions (ds `61418ed4-fb84-48fc-b372-247fbabe0c33`), select à 2 options : **Announce** · **Line-up reveal**. Vide = **Auto**.

### Logique de décision de M1
- `Content Phase = Announce` **et** line-up vide → Vague 1 (Announce).
- `Content Phase = Announce` **et** line-up rempli → **VERROU** : 0 tâche (règle Karter, voir §6).
- `Content Phase = Line-up reveal` → Vague 2 (Reveal), forcée.
- **Vide (Auto)** : line-up vide → Vague 1 ; line-up rempli → Vague 2 directement (cas « line-up complet dès le départ »).

### Idempotence
M1 lit d'abord la checklist existante et **ne recrée jamais un doublon** : à chaque run, il n'ajoute que les tâches de la vague ciblée qui n'existent pas encore. Quand le line-up arrive, on relance → il garde la Vague 1 et ajoute la Vague 2, sans doublon.

## 4. Structure exacte du workflow (par API n8n-mcp)

Trigger **Manual** → chaîne linéaire de 6 nodes :

1. **When clicking Execute workflow** (`n8n-nodes-base.manualTrigger` v1).
2. **Read the pilot edition** (`n8n-nodes-base.notion` v2.2, op `databasePage:get`) — `pageId` en mode URL = `https://www.notion.so/39e24f2fef0c812ab3bbe8636418a482` (garde-fou en dur sur l'ID pilote). Credential `AFTRSN-n8n-Backstage` (`sCsj206WVkNxO0Jl`).
3. **Read existing checklist** (`notion` v2.2, op `databasePage:getAll`, `returnAll:true`, Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5`) — pour l'idempotence.
4. **Build the brief** (`n8n-nodes-base.code` v2) — lit `$('Read the pilot edition').first().json`, garde-fou ID, détecte `hasLineup` (Lineup notes OU relation Lineup), lit `property_content_phase`, calcule `waves` + `blockedReason`, construit l'objet `brief`.
5. **Plan production tasks** (`code` v2) — lit `$('Build the brief')` + `$('Read existing checklist').all()`, catalogue des 2 vagues (5 chantiers chacune), dédoublonne par nom exact, émet uniquement les tâches nouvelles de la/les vague(s) ciblée(s).
6. **Generate production checklist** (`notion` v2.2, op `databasePage:create`, Tasks DB) — `title = {{ $json.taskName }}` ; propriétés : `Status|select`=To Do, `Executor|select`=🤖 Agent, `Category|select`={{ $json.category }} (Brand & Community), `Event|relation` = `{{ [$json.eventId] }}` (⚠️ tableau).

**Settings :** `executionOrder v1`, **Error Workflow = Sentinel `xzaH0uWy0idKVphF`**.

### Catalogue des tâches (nommage, séparateur « · », jamais « — »)
- Announce : `Announce · Copy & captions`, `Announce · Teaser visual (flyer/story, no line-up)`, `Announce · Wix event page (publish)`, `Announce · Newsletter (save-the-date)`, `Announce · Instagram post + story`.
- Line-up reveal : `Line-up reveal · Copy & captions`, `Line-up reveal · Flyer with line-up (flyer/story)`, `Line-up reveal · Wix event page update (artwork + line-up)`, `Line-up reveal · Newsletter (line-up)`, `Line-up reveal · Instagram reveal post + story`.
Chaque nom se termine par ` · {nom d'édition}`.

## 5. Parité Notion (règle visible côté humain)

Champ formule **« Next content wave »** ajouté sur Editions. Il affiche sur chaque fiche ce que M1 fera : `Announce (wave 1)`, `Line-up reveal (wave 2)`, `Line-up reveal (auto)`, `Announce (auto)`, ou **`Announce locked (line-up set)`** quand Announce est choisi alors que le line-up est rempli. (Notion ne peut pas bloquer « en dur » un select selon un autre champ ; la formule assure la parité visuelle. Blocage automatique dur = automation Notion à poser en UI, non fait au MVP.)

## 6. Restriction « verrou Announce » (règle Karter)

Une fois le line-up rempli, on ne peut plus redéclencher l'annonce : `Content Phase = Announce` + line-up présent → `waves = []`, `blockedReason` renseigné, **0 tâche créée**. Le seul levier à ce stade = modifier le line-up (qui rafraîchit la Révélation). Codé côté n8n ET reflété côté Notion (formule).

## 7. Tests bout-en-bout (exécutions réelles, garde-fou création désactivé puis activé)

- **#14170** (Auto, line-up présent) → brief `waves:["reveal"]` → **5 tâches « Line-up reveal · »** créées. ✅ (cas « line-up dès le départ → Vague 2 directe »)
- **#14179** (re-run identique) → Plan output **0**, node de création non exécuté → **0 doublon**. ✅ (idempotence)
- **#14190** (Content Phase = Announce, line-up présent AVANT le verrou) → 5 tâches « Announce · » (état de test).
- **#14201** (Content Phase = Announce, line-up présent, APRÈS le verrou) → brief `waves:[]` + `blockedReason` → **0 tâche**. ✅ (verrou)
Vérifications croisées par requête SQL Notion (Executor 🤖 Agent, Status To Do, relation Event = pilote) OK à chaque étape.

## 8. IDs clés

- Workflow M1 : `git92S6gDU1hGZ2b` · Sentinel : `xzaH0uWy0idKVphF` · cred Notion : `AFTRSN-n8n-Backstage` = `sCsj206WVkNxO0Jl`.
- Notion : Editions ds `61418ed4-fb84-48fc-b372-247fbabe0c33` (DB `dad7f826891f4c2980e4183c59bac8b0`) · Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` / ds `e50f931d-3f8d-483f-895a-c09408455128` · édition pilote Mini Café `39e24f2fef0c812ab3bbe8636418a482`.
- Champs Notion ajoutés : `Content Phase` (select Announce/Line-up reveal) + `Next content wave` (formule).

---

## Difficultés rencontrées

1. **Manual Trigger non déclenchable par API** : l'outil `n8n_test_workflow` ne gère que webhook/form/chat. Chaque run de test dépend d'un clic « Execute » de Karter dans n8n.
2. **Éditeur n8n périmé** : après une modif par API, un clic « Execute » sur un onglet n8n déjà ouvert rejoue l'ancien canvas (brief vide, node resté désactivé) → faux négatifs (runs #14122/#14125).
3. **Forme de sortie du node Notion** : les propriétés ressortent en clés `property_snake_case` (`property_city`, `property_lineup_notes`, `property_content_phase`, `property_date.start`, `name`=titre), pas sous leurs noms d'affichage → premier brief vide.
4. **Champ relation Notion** : passer une chaîne à `relationValue` casse le node (`value.relationValue.filter is not a function`).
5. **Propriété vide masquée dans Notion** : le champ `Content Phase` fraîchement créé n'apparaissait pas sur la fiche (Notion masque les propriétés vides).
6. **Formule non requêtable en SQL** : `Next content wave` est dans `notAvailableInQuerySql` → vérification seulement sur la fiche, pas par requête.
7. **Pas d'archivage de page via l'API dispo** : `notion-update-page` n'expose pas de commande trash → nettoyage des tâches de démo laissé à Karter (multi-select + Delete).

## Solutions implémentées

1. Test par clic Karter + lecture de l'exécution par API (`n8n_executions get`), création désactivée d'abord pour inspecter lecture/brief sans polluer Notion.
2. **Toujours recharger (F5) l'onglet n8n après une modif API, avant de cliquer Execute** (loi à retenir).
3. Code du brief réécrit pour lire les clés `property_*` (mapping explicite), avec référence par nom de node `$('Read the pilot edition').first().json` (robuste au réordonnancement des nodes).
4. `relationValue` en **tableau** via expression `={{ [$json.eventId] }}`.
5. Expliqué le masquage : « Show empty properties » ou donner une valeur ; parité assurée par la formule visible.
6. Vérifications de la formule sur la fiche (visuel Karter), pas par SQL.
7. Nettoyage manuel (2 clics) documenté ; les 5 « Line-up reveal » gardées comme démo.

## Lessons learned

1. **n8n exécute le canvas ouvert, pas forcément le sauvegardé** : après tout build/patch par API, recharger l'éditeur est obligatoire avant un run manuel. Sinon on débogue une version fantôme.
2. **Le node Notion (lecture) sort les propriétés en `property_snake_case` + `name` pour le titre** : toujours mapper ces clés, pas les noms d'affichage.
3. **Relation Notion en écriture = tableau d'IDs** (`[$json.eventId]`), jamais une chaîne.
4. **Un premier run à blanc (node d'écriture désactivé) puis lecture de l'exécution par API** est la boucle de debug la plus propre pour un trigger manuel.
5. **L'idempotence par lecture préalable + dédoublonnage par nom** est ce qui rend un orchestrateur « relançable » sans dégât : essentiel dès qu'une info (line-up) arrive en plusieurs temps.
6. **Modéliser l'info tardive comme une PHASE de contenu, pas comme un blocage** : le retard du line-up devient un deuxième temps fort de communication. Un champ de contrôle Notion (Auto/forçage) + un verrou métier (pas de retour arrière) donnent à l'humain la main tout en gardant la cohérence.
7. **Parité humain/automate** : quand une règle vit dans le workflow, l'exposer aussi dans Notion (formule) évite les incohérences de perception, même si Notion ne peut pas l'imposer « en dur ».
8. **Règle de style Karter** : jamais de « — » dans ce que je génère (séparateur « · »). Le « — » présent dans le nom d'édition est sa donnée, conservée telle quelle.
