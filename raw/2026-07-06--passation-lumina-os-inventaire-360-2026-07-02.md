---
type: raw
title: "PASSATION_LUMINA-OS_Inventaire-360_2026-07-02"
source_url: "drive:147VXIv39pn0m43-G8y719ir9Xy2ejSE6"
captured: 2026-07-06
vault: brands
brand: AFTRSN
immutable: true
---

# PASSATION — Construire la « Bible 360° » de LUMINA OS (inventaire complet)

**Date :** 2026-07-02 · **Auteur :** Claude (Cowork) pour Karter (Katel, CEO AFTER SUN PEOPLE) · **Destination :** amorcer une **nouvelle conversation** dédiée à l'inventaire.

> **But de la prochaine session :** produire **UN document de référence absolu** (« la Bible ») donnant une vue 360° de tout LUMINA OS — chaque catégorie, chaque workflow, chaque outil interne, chaque IA, leur rôle, leur criticité, et leur statut publié/non-publié avec la raison. Règle d'or à inscrire dans la Bible : *avant de toucher à quoi que ce soit, on lit la Bible pour savoir quoi toucher, quoi ne pas toucher, et pourquoi.*

---

## 0. Ce qu'est LUMINA OS (en clair)

Écosystème d'automatisation & d'agents IA construit sur **n8n auto-hébergé** (`n8n.aftersunpeople.com`, Hetzner + Coolify), **brand-agnostic** : *1 socle, N marques*. Trois piliers :
1. **Mémoire** — connaissance vectorielle (pgvector) par marque + partagée, alimentée depuis Google Drive → GitHub (source de vérité) → pgvector/Notion.
2. **Cerveau** — un orchestrateur (Maestro) + des sous-agents spécialisés, un **routeur multi-LLM** (choisit le modèle selon la tâche), un **exécuteur apprenant** (Hermes).
3. **Chaîne de traitement** — workflows rangés en **étapes numérotées** (process-flow) : entrée → mémoire → cerveau → automation → connexions.

Marque pilote : **AFTER SUN PEOPLE (AFTRSN)**. Objectif business : proposer ces automatisations à de petites entreprises.

---

## 1. Structure attendue de la Bible (plan du document à produire)

La nouvelle session devra générer un `.md` (+ éventuellement `.xlsx` récap) avec **ces sections** :

**A. Résumé exécutif 360°** — schéma de la chaîne (INTAKE → MEMORY → BRAIN → AUTOMATION → CONNECT), en une page.
**B. Glossaire** — Lumina OS, marque, banque `<code>_memory`, collection, agent, sous-agent, routeur, Hermes, skill, canon, épisode, insight.
**C. Les catégories (dossiers-étapes)** — pour chacune : nom, rôle dans la chaîne, « où chercher si problème ».
**D. Inventaire par catégorie** — sous chaque catégorie, la liste des workflows.
**E. Fiche détaillée par workflow** (le cœur), avec pour chaque :
  - Nom exact · ID n8n · catégorie/dossier · tags (4 axes).
  - **Statut : publié ou non + POURQUOI** (voir §5).
  - Déclencheur (webhook / scheduled / event / manuel / sous-workflow appelé).
  - **Description du rôle** (à quoi il sert dans la chaîne).
  - **Outils / nodes internes développés** : lister les nodes clés (Code, HTTP, Postgres, AI Agent, outils `Call`, Knowledge…) et **ce que chacun fait**.
  - **Dépendances** amont (qui l'appelle / d'où vient la donnée) et aval (ce qu'il alimente).
  - **Criticité** (🔴 critique / 🟠 important / 🟢 utilitaire) + « quoi ne pas toucher sans précaution ».
  - Credentials utilisées.
**F. Tableau récapitulatif des fonctionnalités** — colonnes : `Workflow | Outil/fonction interne | Ce que ça fait | Criticité`.
**G. Les IA / LLM implémentées** — lesquelles, où, quel rôle, via quelle passerelle (voir §4).
**H. Composants transverses** — tables pgvector, collections, MCP, intégrations externes (GitHub, Notion, Drive), credentials.
**I. Tableau publié / non-publié** — chaque workflow, statut, **raison** (PROD ? test ? déprécié ? sous-workflow appelé donc pas besoin d'être « actif » ?).
**J. Criticité & garde-fous** — la liste des invariants (PII jamais au cloud, actions critiques gatées, embedding figé, Git = vérité, n8n fait foi).
**K. Renvois** — vers les POS détaillés (§7) pour chaque brique.

---

## 2. État réel déjà collecté (point de départ — À VÉRIFIER puis enrichir)

**Projet n8n unique** (personnel) `Tn1aNTcuxmqqHnKU`. **2 racines-marque** : `01-LUMINA` (partagé, tag 🌐 SHARED) et `02-AFTRSN` (marque, tag 🌞 AFTRSN). **32 workflows : 21 publiés, 11 non publiés.**

### Catégories (dossiers-étapes, standard process-flow)
`01-INTAKE` 📥 (entrée données) → `02-MEMORY` 🧠 (connaissance) → `03-BRAIN` 🤖 (agents + `Sub-Agents`, routeur 🧭, exécuteur ⚙️) → `04-AUTOMATION` 🔄 → `05-CONNECT` 🔌 → `08-UTIL` 🛠️ ; hors chaîne : `Z_ARCHIVES` ⚫, `SandBox` 🔵. (Détail : POS Classification process-flow.)

### Workflows LUMINA (partagés 🌐)
| Workflow | Catégorie | Publié | Rôle (1 ligne) | Criticité |
|---|---|---|---|---|
| LUMINA-INTAKE-ROUTER/DRIVE→GITHUB | 01-INTAKE | ✅ | Draine la Lumina Inbox, classe (LLM) chaque fichier, écrit dans GitHub, supprime après confirmation | 🔴 |
| LUMINA-INTAKE-ARCHIVE/RAW→DRIVE | 01-INTAKE | ✅ | Archive les fichiers bruts vers Drive | 🟠 |
| LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR | 02-MEMORY | ✅ | Ingère le wiki GitHub → pgvector (chunk/embed/insert) | 🔴 |
| LUMINA-MEMORY-INGESTION/PUBLISH-NOTION | 02-MEMORY | ✅ | Synchronise le wiki vers Notion (upsert idempotent) | 🟠 |
| LUMINA-MEMORY-INGEST-TEXT | 02-MEMORY | ✅ | Ingestion texte/markdown réutilisable → `<brand>_memory` (sous-workflow) | 🟠 |
| LUMINA-RETRIEVAL-MEMORY/WEBHOOK | 02-MEMORY | ✅ | Recherche vectorielle (memory-search), filtre `collection` | 🔴 |
| LUMINA-MEMORY-WRITE/WEBHOOK | 02-MEMORY | ✅ | Écrit un épisode/mémoire (Save Episode) (sous-workflow) | 🟠 |
| LUMINA-MEMORY-CONSOLIDATION/NIGHTLY | 02-MEMORY | ✅ | Cron 03:00 : condense épisodes → insights (+ auto-promo skills) ; incrémentale + dédup | 🟠 |
| LUMINA-BRAND-PROVISION | 02-MEMORY | ✅ | Provisionne une marque (banque + `memory_registry` + `brand_profiles`) | 🟠 |
| LUMINA-RETRIVIAL-MEMORY (MANUAL/TEST) | 02-MEMORY | ❌ | Version test de la recherche | 🟢 |
| LUMINA-dédup / idempotence | 02-MEMORY | ❌ | Brique dédup (remplacée par ON CONFLICT) | 🟢 |
| LUMINA-AI-Router | 03-BRAIN | ✅ | Routeur multi-LLM par `task_type` (OpenRouter), garde-fou PII, injection RAG (sous-workflow) | 🔴 |
| LUMINA-Hermes-Exec | 03-BRAIN | ✅ | Bras d'exécution apprenant (skills) ; appelle Hermes Ops (cookie-auth) (sous-workflow) | 🟠 |
| LUMINA-ORCHESTRATOR-SYNC | 04-AUTOMATION | ✅ | Orchestration/synchro automatisée | 🟠 |
| LUMINA-MCP-Server | 05-CONNECT | ✅ | Serveur MCP exposant `search_brand_memory` aux clients externes | 🟠 |
| LUMINA-UTIL-CONSOLE SQL (POSTGRES) | 08-UTIL | ❌ | Console SQL manuelle | 🟢 |
| LUMINA-UTIL-SQL-CONSOLE | 08-UTIL | ❌ | Console SQL (variante/one-off) | 🟢 |
| LUMINA-DDL-RUNNER (manuel, one-off) | 08-UTIL | ❌ | Exécuteur DDL ponctuel | 🟢 |

### Workflows AFTRSN (marque 🌞)
| Workflow | Catégorie | Publié | Rôle | Criticité |
|---|---|---|---|---|
| AFTRSN-Maestro | 03-BRAIN | ✅ | Orchestrateur de la marque (hub) ; 8 outils : Culture, Experience, Secretary, Marketing, Comptable, Qualite, Router, Save Episode | 🔴 |
| AFTRSN-Culture-Steward | 03-BRAIN/Sub-Agents | ✅ | Challenger crédibilité culturelle | 🟠 |
| AFTRSN-Experience-Designer | Sub-Agents | ✅ | Challenger design expérientiel | 🟠 |
| AFTRSN-Secretary | Sub-Agents | ✅ | Mails draft-only, agenda, fichiers (+ Hermes Ops) | 🟠 |
| AFTRSN-Marketing | Sub-Agents | ✅ | Marketing events + garde-voix de marque (+ Hermes Ops) | 🟠 |
| AFTRSN-CHANNEL-CONTENT | Sub-Agents | ✅ | Contenu réseaux sociaux + newsletter | 🟠 |
| AFTRSN-Comptable-Finance | Sub-Agents | ❌ | Budgets/coûts/marges/cash-flow (n'engage jamais de dépense) | 🟠 |
| AFTRSN-Qualite-Compliance | Sub-Agents | ❌ | Challenger qualité/conformité (verdict/risques/fixes), veille PII | 🟠 |
| AFTRSN-MEMORY-INGESTION/DRIVE→AFTRSN_MEMORY | 02-MEMORY | ✅ | Ingestion de la bible marque → `aftrsn_memory` | 🟠 |
| AFTRSN-MEMORY-INGESTION/DRIVE-RECURSIVE | 02-MEMORY | ✅ | Ingestion récursive Drive (PDF + markdown) | 🟠 |

### Archives / bac à sable (non publiés, normaux)
`ZZZ_20260623_Archive-AFTRSN-Ingestion-Knowledge-Bank` (déprécié) · `ZZZ_20260623_LUMINA-INGESTION NOTION (OBSOLET)` (déprécié) · `ZZZ_20260702_tmp_hermes_reach_test` (test temporaire) · `Test-3-AI` (SandBox).

---

## 3. ⚠️ Anomalie à traiter dans l'inventaire

**AFTRSN-Comptable-Finance** et **AFTRSN-Qualite-Compliance** sont **non publiés**, alors que les 5 autres sous-agents le sont. Ils restent appelables par Maestro (executeWorkflowTrigger n'exige pas « actif »), mais par cohérence il faudra décider : **les publier** ou documenter explicitement pourquoi non. → À trancher et noter dans la Bible.

---

## 4. IA / LLM implémentées (à détailler)

| Moteur | Via | Où | Rôle |
|---|---|---|---|
| Claude Sonnet | OpenRouter `~anthropic/claude-sonnet-latest` | Maestro, Marketing, Channel-Content, Comptable, Qualite | Rédaction / raisonnement / orchestration |
| Gemini Flash | `~google/gemini-flash-latest` | Culture-Steward, Experience-Designer | Recherche / synthèse rapide & bon marché |
| GPT-mini | `~openai/gpt-mini-latest` | Secretary | Tâches légères |
| OpenAI `text-embedding-3-small` (1536) | API OpenAI | Toute la mémoire (ingestion + recherche) | Embeddings (figé, ne pas changer) |
| OpenAI `gpt-4o-mini` | API OpenAI | Intake Router | Classifieur des fichiers entrants |
| Hermes (Nous, self-hosted Coolify) | `hermes.aftersunpeople.com` | LUMINA-Hermes-Exec | Exécuteur Ops (recherche web/exécution) ; futur modèle local |

Le **Router** (`Cu8SozYmondKM8RB`) choisit le modèle selon `task_type` (voir POS Routeur + benchmark xlsx v2).

---

## 5. Publié vs non-publié — grille de lecture

- **Publié (actif)** = tourne sur son déclencheur (webhook/cron/event) OU est un sous-workflow appelé qu'on a choisi d'activer → **en PROD**.
- **Non publié** peut être **légitime** : (a) *test* (🔵 TEST) ; (b) *déprécié* (⚫ `ZZZ_`) ; (c) *utilitaire manuel* (console SQL, DDL) qu'on n'active pas ; (d) *sous-agent appelable* pas encore publié (cas Comptable/Qualité — à trancher).
- La Bible doit donner **la raison** pour chacun (pas juste le statut).

---

## 6. Comment collecter le détail « outils internes » (méthode pour la prochaine session)

Le détail des **nodes/outils dans chaque workflow** se lit par l'**API interne n8n** depuis la session Chrome de Karter (rien de destructif) :
```js
const hdr={credentials:'include',headers:{'browser-id':localStorage.getItem('n8n-browserId'),'accept':'application/json'}};
const w=(await (await fetch('/rest/workflows/<ID>',hdr)).json()).data;
w.nodes.map(n=>n.name+' ['+n.type.split('.').pop()+']');   // liste des outils/fonctions du workflow
```
Pour chaque workflow : lister les nodes, décrire ce que fait chacun (Code = quelle logique, HTTP = quel endpoint, Postgres = quelle requête, AI Agent = quel rôle, `Call '…'` = quel sous-agent, Knowledge = quelle banque). ⚠️ L'affichage peut filtrer les gros dumps SQL/tokens → extraire par petits morceaux / booléens si besoin.

---

## 7. Sources documentaires à lire (déjà produites, projet Claude + Drive + Lumina Inbox)

- **PLAN-PROJET_LUMINA-AI-OS** (v3.1) — état consolidé.
- **POS Classification workflows n8n (process-flow)** — catégories + 4 axes de tags + registre des ids.
- **POS/MILESTONE câblage Maestro↔sous-agents↔Router** · **POS Routeur multi-LLM** + benchmark xlsx v2.
- **POS mémoire vivante** (épisodique + consolidation + RAG) · **POS ingestion multi-format**.
- **POS bibliothèque de skills apprenante** · **POS Hermes outil Ops (cookie-auth)**.
- **POS audit/édition n8n par API interne** · **LUMINA-PLAYBOOK v1** · **POS clonage roster (cloneRoster)** · **POS onboarding nouvelle marque**.
- **Mémoire persistante** (index `MEMORY.md`) : agents état n8n, mémoire vivante, playbook multimarques, classification, chronologie documentaire.

---

## 8. Invariants (garde-fous — à rappeler dans la Bible)

1. `task_type='private'` (PII) → **jamais** au cloud.
2. Actions critiques (dépense, envoi, publication, suppression) → **validation humaine**, quel que soit le niveau d'un skill.
3. Embedding **figé** `text-embedding-3-small` 1536 — ne pas changer sans tout ré-encoder.
4. **Git = source de vérité** ; pgvector/Notion = dérivés régénérables (jamais édités à la main).
5. **n8n fait foi sur la doc** en cas d'écart — vérifier le workflow live avant d'agir.
6. **Suppressions/DROP = réservés à Karter** (l'assistant ne fait pas d'action destructive).

---

## 9. Premier message suggéré pour la nouvelle conversation

> « Construis la Bible 360° de LUMINA OS à partir de la passation `PASSATION_LUMINA-OS_Inventaire-360_2026-07-02.md`. Ouvre chaque workflow n8n (API interne) pour lister ses nodes/outils et ce que chacun fait, complète les fiches (§1-E), le tableau récap (§1-F) et la grille publié/non-publié (§1-I), puis livre un `.md` (+ `.xlsx` récap) double-sauvé. »

---

*PASSATION — 2026-07-02, Claude (Cowork). Données d'état vérifiées en live (32 workflows, 21 publiés / 11 non). À utiliser comme point de départ de l'inventaire ; tout est à confirmer node par node dans la nouvelle session.*
