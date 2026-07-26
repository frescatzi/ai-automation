---
type: raw
title: "AFTRSN_Architecture_Processus_20260704"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn_architecture_processus_20260704.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# AFTRSN — Architecture par processus métier

**Statut :** Draft v1 — 04.07.2026 — à valider par Karter avant toute construction dans n8n.
**Remplace :** `W1_Decoupage_Sous-Workflows_20260703.md` (découpage technique W-1a→W-1g, abandonné : il découpait le workflow, pas le métier).
**Principe directeur (Karter, 04.07.2026) :** on construit directement les processus. Des processus simples, efficaces, efficients, lisibles un par un sur le long terme. La réalité montrera les optimisations à opérer — pas de restructuration théorique préalable.
**Approche retenue :** reprendre l'existant, retirer ce qui est obsolète, ajouter ce qui est fonctionnel — pas de reconstruction à zéro.

---

## 1 · La chaîne métier de référence

Le cycle de vie d'une édition, tel que dicté par Karter le 03.07.2026 :

```
Date confirmée par le lieu
   │
   ▼
[P1] Création d'édition ──────── automatique, sans validation humaine
   │
   ▼
[P2] Pack promo ───────────────── humain (Canva + core team)
   │
   ▼
[P3] Publicités ───────────────── auto-brouillon + revue humaine + Hermès
   │
   ▼
[P4] Site web & Newsletter ────── auto-construction + publication manuelle
   │
   ▼ (l'événement a lieu)
[P5] Post-event ───────────────── médias, debrief, closing budget
```

Un processus = un workflow n8n (ou une activité humaine outillée). Pas de sous-découpage technique à l'intérieur d'un processus tant que la réalité n'en montre pas le besoin.

**Chaîne d'exécution (décision Karter, 04.07.2026) :** Maestro **orchestre et délègue** — il n'exécute rien lui-même. Toute action automatique (piloter Google Drive, Notion, Calendar, Wix, les plateformes publicitaires…) est **exécutée par Hermès** (plateforme d'exécution partagée, passerelle n8n `LUMINA-Hermes-Exec`, gouvernance intégrée au code : skills `[autonomous]` exécutent le non-critique, `[supervised]` proposent seulement, action critique → validation Karter sans exception).

```
Karter / date confirmée
   → Maestro (orchestration, délégation)
      → Hermès (exécution des actions automatiques)
         → Outils : Drive, Notion, Calendar, Wix, pubs…
```

**Mode d'exécution retenu (décision Karter, 04.07.2026) : option (b)** — Hermès pilote directement les outils avec ses propres skills/runbooks, **sous supervision de Maestro**. Les workflows n8n existants (ex. `AFTRSN-Event-Creator`) servent de référence de migration vers la bibliothèque Skills.

**Cycle de supervision Maestro ↔ Hermès :**

1. **Délégation** — Maestro confie la tâche à Hermès avec l'objectif et les **critères d'acceptation** (indispensables : sans eux, le contrôle de l'étape 4 reste subjectif).
2. **Exécution** — Hermès sélectionne ses skills et exécute. Gouvernance inchangée : `[autonomous]` exécute le non-critique, `[supervised]` propose seulement, action critique → validation Karter sans exception.
3. **Rapport** — Hermès rend le travail + le rapport d'exécution : résultat et **skills utilisés** (déjà journalisé par `LUMINA-Hermes-Exec` : `Build Log → Log Skills`).
4. **Contrôle** — Maestro (ou le spécialiste concerné) vérifie le rendu contre les critères et la cohérence avec les banques de données. Si une connaissance manque en banque, on la complète.
5. **Issue :**
   - **Accepté** → Maestro écrit dans la banque de données : ce qui a été fait, avec quels skills — visible par tous les agents.
   - **Rejeté** (par Maestro ou un spécialiste) → diagnostic : problème de connaissance ou skill en cause ? Le rejet et sa cause sont **notés quand même** (l'échec nourrit la boucle). Arbitrage Karter au-delà d'un aller-retour, pour éviter les boucles infinies entre agents.
6. **Amélioration continue** — les agents peuvent pousser leurs connaissances/skills vers Hermès pour qu'il prenne connaissance et fasse mieux ; la bibliothèque Skills s'enrichit des succès comme des rejets.

## 2 · Détail par processus

### P1 — Création d'édition
**Règle métier :** une date reçue du lieu = un événement = déclenchement automatique complet de l'administratif, sans validation humaine, identique à chaque fois.
**Workflow :** `AFTRSN-Event-Creator` — **existant, on le garde tel quel** (monolithe corrigé le 04.07 : calendrier → fiche Notion, DB repointées 🎭, Merge2 supprimé).
**Ce qu'il produit :** upsert Venue → fiche Edition (hub central) → checklist → arborescence Drive (Année/Mois/Édition, la date groupe, jamais la ville) → événement Calendar → lignes budget → épisode mémoire.
**Budget :** auto-brouillon — lignes pré-remplies à terme avec les moyennes historiques + la capacité du lieu (champ `Capacity` de la page Venue 🎭) ; finalisation toujours humaine. Aujourd'hui lignes vides, enrichissement quand la DB 🎭 aura de l'historique.
**Note :** le dossier photos & vidéos créé ici est une coquille vide remplie en P5 (usage post-promo) — à ne pas confondre avec le pack promo (P2).
**État : ✅ construit et découpé (04.07.2026)** — `AFTRSN-Event-Creator` est devenu un **orchestrateur** de 9 nodes qui enchaîne 6 sous-workflows métier (décision Karter : vue simple et limitée par étape). Emplacement (rangement Karter du 04.07.2026) : `Personal / 02-AFTRSN / AFTRSN-02-BUSINESS AUTOMATION / AFTRSN-PROCESS-01-Event-Creation` — les 7 workflows du processus y vivent ensemble :

| Sous-workflow | ID n8n | Contenu |
|---|---|---|
| AFTRSN-P1.1 — Validation & Lieu | `bR9zQRxhcPYvNRjS` | validation input, normalisation, upsert Venue |
| AFTRSN-P1.2 — Dossiers Média | `bOd0PENXx7xMyD2a` | arborescence Drive complète |
| AFTRSN-P1.3 — Fiche Edition | `3MAoAeG1EHklFLeG` | page Notion + injection checklist |
| AFTRSN-P1.4 — Calendrier | `aO7JVUD7F4JoFrjD` | événement Google Calendar |
| AFTRSN-P1.5 — Budget | `1TqTmFSnJvIZXTQq` | 5 lignes Budget & Finance |
| AFTRSN-P1.6 — Clôture & Mémoire | `m8N5RId1TgeLCmvU` | épisode mémoire + résumé final |

Chaque sous-workflow reçoit uniquement ses paramètres (plus aucune référence croisée `$('Node')` entre blocs), se teste seul, et l'échec de la validation lieu arrête tout (`Lieu OK ?` → `Erreur - reponse`). Au passage, correction d'un bug latent : `Create folder1` lisait `media_date_prefix` depuis la sortie Drive au lieu des données d'entrée (origine probable des dossiers « _ » dupliqués) — désormais lu depuis le trigger. **Test end-to-end validé le 04.07.2026** (payload jetable ZZTEST-EP99) : chaîne complète en ~15 s, les deux chemins de l'upsert Venue vérifiés, checklist injectée, calendrier → fiche Notion, 5 lignes budget, épisode mémoire, résumé final avec URLs réelles pour Maestro. Données de test intégralement purgées (Notion, Drive, Calendar) et **workflows publiés le 04.07.2026** (« v1.0 — P1 Event-Creation », tags 🌞🔄🟢 PROD). **P1 est en production.**
**Invocation cible :** aujourd'hui Maestro appelle ce workflow directement ; cible = Maestro → Hermès → exécution directe des outils (option b, voir §1). Le workflow n8n validé sert de **référence de migration** : sa logique métier (upsert, arborescence, checklist, contrats de données) devient un runbook de la bibliothèque Skills d'Hermès. On termine d'abord son test — un runbook se migre depuis une référence qui marche.

### P2 — Pack promo
**Règle métier :** flyer + déclinaisons = travail humain sur Canva, validé par la **core team**, diffusé manuellement. Jamais automatisé.
**Workflow :** aucun pour le contenu. Automatisation possible plus tard, limitée au support : rappels d'échéance basés sur la date d'édition, suivi d'état dans la checklist de la fiche Edition.
**État : ⚪ processus humain — rien à construire pour l'instant.**

### P3 — Publicités
**Règle métier :** campagnes auto-créées avec paramètres quasi identiques (seule la ville change, Zürich = référence). Jamais publiées sans revue humaine. Une fois live : monitoring + propositions d'optimisation par un agent via **Hermès**, exécution uniquement après validation Karter.
**Workflow :** à construire (correspond au volet publicitaire de W-4 Marketing-Planner dans la roadmap). Prérequis : runbook « optimisation publicitaire » dans la bibliothèque Skills d'Hermès (point ouvert de la passation du 03.07).
**État : 🔲 à construire — Étape 3 de la roadmap (semaine du 16–22.07).**

### P4 — Site web & Newsletter
**Règle métier :** page événement Wix auto-construite, publication manuelle (bascule auto possible plus tard, au choix de Karter). Newsletter déclenchée à la publication du site, uniquement si un segment de contacts existe pour la ville (toujours le cas pour Zürich). Segmentation aujourd'hui manuelle dans Wix — son automatisation est explicitement un futur workflow séparé, hors périmètre.
**Workflow :** à construire (volet contenu/diffusion de W-4).
**État : 🔲 à construire — Étape 3 de la roadmap.**

### P5 — Post-event
**Règle métier :** remplissage du dossier photos & vidéos (usage post-promo), debrief, capture de connaissance, closing budgétaire.
**Workflow :** W-7 Knowledge-Capture (ingestion debriefs → `aftrsn_memory`, collection `insights`) + closing budget (rapport financier, volet closing de W-6).
**État : 🔲 à construire — Étape 4 de la roadmap (août).**

## 3 · Processus transverses (hors chaîne, en soutien)

| Processus | Workflow | Rôle | État |
|---|---|---|---|
| Coordination lieux | W-2 Venue-Coordinator | outreach venues, follow-ups J+3/J+7 → aboutit à la date confirmée qui déclenche P1 | 🔲 Étape 2 |
| Coordination artistes | W-3 Artist-Coordinator | outreach booking, follow-ups, brief D-7 | 🔲 Étape 2 |
| Réponse email | Email-Responder (`AFTRSN-PROCESS-02`) | lecture Gmail horaire → drafts via Secretary → notif Telegram détaillée → mémoire ; jamais d'envoi automatique | 🟢 **en production depuis le 04.07.2026** |
| Suivi budget | W-6 Budget-Tracker | lecture continue, alertes seuils, rapport de closing (utilisé par P5) | 🔲 Étape 2 |
| Exécution agents | Hermès (plateforme partagée) | monitoring/optimisation pubs (P3) ; gouvernance : dépense/envoi/publication/suppression → validation Karter obligatoire | ✅ existant |

## 4 · Ce qui est repris de l'ancien découpage, ce qui est retiré

**Retiré (obsolète) :**
- Le schéma W-1a→W-1g et son orchestrateur : découpage technique du workflow, pas des processus métier. Source de confusion.
- La dépendance `folder_id` → calendrier : supprimée à la racine le 04.07 (le calendrier pointe vers la fiche Notion, hub central).

**Repris (toujours valide) :**
- Le périmètre métier de W-1 (administratif automatique ; promo/pubs/site/newsletter en aval, gatés humain).
- Les règles de gestion d'erreur : pas d'écriture partielle silencieuse, aucune action destructive, `{status:"error", step, message}` en sortie d'échec.
- Les invariants : filtre Notion « Contains » (jamais « Equals » sur un titre), PATCH pour l'injection checklist, HTTP Request pour Claude, mémoire épisodique à chaque run significatif.
- Les DB cibles 🎭 : Venues `b9001abd-8332-4bbb-9673-6deb5159b4f3`, Editions `dad7f826-891f-4c29-80e4-183c59bac8b0`, Budget & Finance `05d7f280-5ffa-48c4-a1b5-a32dc4c38c99`.

## 5 · Règles de construction (valables pour tous les processus)

1. Vue simple et limitée par étape (règle Karter, 04.07.2026) : un gros processus est découpé en **sous-workflows métier** appelés par un **orchestrateur** — quand on travaille, on est directement dans le bon workflow et on sait ce qu'on regarde. Chaque sous-workflow reçoit uniquement ses paramètres, jamais de référence croisée `$('Node')` entre blocs.
2. **Maestro délègue, Hermès exécute** : toute action automatique passe par Hermès, quel que soit l'outil piloté (Drive, Notion, Wix…). Maestro n'exécute jamais directement.
3. Toute numérotation commence à 1, jamais à 0 (règle Karter, permanente).
4. Toute action critique (dépense, envoi, publication, suppression) est gatée par validation humaine — l'administratif de P1 est la seule exception, voulue et bornée (actions non critiques uniquement : créations de dossiers/pages, compatibles skill `[autonomous]`).
5. Chaque run significatif journalise un épisode dans `aftrsn_memory` via `LUMINA-MEMORY-WRITE/WEBHOOK`.
6. Convention n8n (rangement Karter, 04.07.2026) : un processus = un dossier `AFTRSN-PROCESS-<nn>-<Nom>` sous `AFTRSN-02-BUSINESS AUTOMATION`, contenant l'orchestrateur et ses sous-workflows ; l'infrastructure vit sous `AFTRSN-01-INFRASTRUCTURE`. Tags Bible §C, nodes renommés clairement, payload jetable `ZZTEST-…` avant toute donnée réelle.

## 6 · Prochaines étapes

1. Validation de ce document par Karter.
2. ~~Trancher le point d'implémentation Hermès~~ → **tranché le 04.07.2026 : option (b)**, avec cycle de supervision Maestro ↔ Hermès (§1).
3. Finir le test end-to-end de P1 (`ZZTEST-EP99`) — en attente du partage 🎭 The Backstage ↔ intégration `AFTRN-n8n` dans Notion.
4. Nettoyer le payload de test, publier P1.
5. Reprendre la roadmap : W-5 Email-Responder (Étape 1), puis Étape 2.

**Portée LUMINA OS :** la règle « Maestro délègue, Hermès exécute » concerne l'architecture du système LUMINA OS lui-même, pas seulement AFTRSN — elle mérite d'être poussée vers le dépôt canonique (repo `wiki/`, collection `canon`), conformément au SYSTEM-CANON §5 (episodic ≠ canon). Un document de session seul ne corrige pas durablement la connaissance de fond du système.
