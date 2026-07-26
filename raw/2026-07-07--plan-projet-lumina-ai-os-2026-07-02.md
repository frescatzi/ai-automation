---
type: raw
title: "PLAN-PROJET_LUMINA-AI-OS_2026-07-02"
source_url: "drive:1xF7ikBpwmg2nvmkBotDkTe98yNgu9Ico"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# PLAN DE PROJET — LUMINA AI OS (v3.1, état consolidé au 2026-07-02)
**Sources :** plan 01-07 + passation 02-07 + audit n8n live 02-07 + **docs recouvrés du 29-06→01-07** (dossiers « Claude Lessons » et « Lumina OS »).

**Vision :** plateforme multi-agents & multi-LLM sur n8n, brand-agnostic (1 workflow, N marques) : mémoire vectorielle multi-banques (pgvector), routage LLM par `task_type` (OpenRouter), bras d'exécution (Hermes Agent), Git = source de vérité documentaire. Marque pilote : **AFTER SUN PEOPLE (AFTRSN)**. Volet 2 (long terme) : revendre la stack comme offre d'agence.

Légende : ✅ fait · 🟡 en cours · ⬜ à faire · ⏸️ différé

---

## Organisation d'agents — VERSION ABSOLUE VALIDÉE (v2 du 01-07, 18:35)

**Vague 1 (5 agents, câblée en PROD)** : Maestro (orchestrateur, entrée/sortie unique, expert curation/événementiel) · Culture & Community Steward (challenger culturel) · Experience & Event Designer (challenger expérientiel) · Secrétaire (mails draft-only, agenda, fichiers) · Marketing events (+ **garde-voix de marque**). **Hermes = outil Ops, pas un agent.**

**Vague 2 (différée)** : Comptable & finance · Qualité & compliance · Content (Insta/newsletter/WhatsApp — base : l'actuel Channel-Content, déjà actif en préfiguration).

**Retiré définitivement** : Music & Sound Curator (pas de connaissance DJ fiable dans un LLM).
**Anciens agents** : Acquisition-Performance → devenu Marketing · Brand-Guardian → archivé (garde-voix = Marketing) · Channel-Content → conservé (contenu réseaux sociaux + newsletter).

**Collaboration (décision 30-06)** : orchestré + appels ciblés — Maestro chef d'orchestre et point de compte rendu unique ; agents appelables entre eux via Call n8n Workflow Tool (chaîne courte, profondeur plafonnée) ; fan-out parallèle ; **Karter = autorité finale sur toute action critique** (dépense, envoi, publication, suppression). Maillage peer-to-peer rejeté (coût/traçabilité).

**Mémoire** : Maestro → `lumina` + lecture `aftrsn` (`brand` via `$fromAI`, défaut lumina) ; les 4 autres + Channel-Content → `aftrsn` fixe.
**Modèles cibles (doc v2, à câbler — cf. A7)** : Maestro + Marketing → `~anthropic/claude-sonnet-latest` · Culture + Experience → `~google/gemini-flash-latest` · Secrétaire → `~openai/gpt-mini-latest` (via credential « OpenRouter account »).

---

## Hermes — exécuteur APPRENANT (décision 02-07)

Hermes reste l'outil Ops appelé par les agents (pas d'agent de plus au roster), mais devient un **exécuteur apprenant**, en 3 couches réutilisant l'architecture existante :

1. **Contexte** : Hermes interroge Knowledge (lumina + aftrsn) avant chaque exécution → exécutions brand-aware.
2. **Skills réutilisables** : chaque procédure exécutée et validée = runbook/skill versionné dans Git → pgvector (`collection='skills'`), consulté par Hermes avant d'agir. Bibliothèque brand-agnostic par construction.
3. **Apprentissage** : mémoire épisodique (WRITE par exécution) + consolidation nocturne = « il étudie comment on travaille » ; long terme : LoRA périodique du modèle local (Phase 4) pour figer les acquis.

**Garde-fou** : autonomie progressive — niveau de maturité par skill (supervisé → autonome selon le track record) ; les actions critiques restent gatées par Karter.

## Playbook agentique multi-marques (décision 02-07)

Objectif : constituer une **bibliothèque de stratégies agentiques réutilisables**. AFTRSN = pilote ; marque 2 = config + adaptation, pas une reconstruction. Livrable : **LUMINA-PLAYBOOK v1** = roster template (orchestrateur + challengers + fonctions business + Ops) + blocs de prompts paramétrés par `brand_profile` + câblage type (Knowledge, Call-tools, Router, triggers) + garde-fous (validation humaine, PII) + skills Hermes génériques. Les POS GÉNÉRIQUES existants en sont l'embryon.

---

## Phase 0 — Infrastructure ✅ COMPLÈTE
- ✅ n8n self-hosté `n8n.aftersunpeople.com` (Hetzner VPS + Coolify)
- ✅ PostgreSQL + pgvector (HNSW cosine)
- ✅ Credentials OpenAI / Anthropic / Gemini + OpenRouter account
- ✅ Hermes Agent sur Coolify → `https://hermes.aftersunpeople.com` (WebUI 8787, mdp var Coolify)

## Phase 1 — Mémoire vectorielle
- ✅ Pipeline GitHub→pgvector (chunk → `text-embedding-3-small` 1536 → INSERT paramétré) ; réparé le 29-06
- ✅ Multi-banques : `memory_registry` + `lumina_memory` (partagée, ex-`asp_memory`) + `aftrsn_memory` (marque)
- ✅ Récupération webhook + MCP `search_brand_memory(brand, question)`
- ✅ Bible de marque ingérée (`aftrsn_memory`, `collection='canon'`)
- ✅ **Mémoire épisodique (02-07)** : webhook/sous-workflow `LUMINA-MEMORY-WRITE` (`zu4jfZbmDz8trQLl`, executeWorkflowTrigger — les webhooks créés par API ne s'enregistrent pas en mode queue) ; Maestro écrit 1 épisode/requête via l'outil `Save Episode` (`collection='episodic'`). Testé : Maestro décide un thème → épisode écrit → retrouvable.
- ✅ **Consolidation nocturne (02-07)** : `LUMINA-MEMORY-CONSOLIDATION/NIGHTLY` (`dVZyCwGYqnqMpS3P`, cron `0 3 * * *`) → lit épisodes récents → LLM condense → embed → INSERT `collection='insights'` (garde tout). Testé : insight « Insights <date> » écrit + récupérable.
- ✅ **Filtrage `collection` dans la recherche (02-07)** : `memory-search` accepte `collection` (sinon exclut `episodic` par défaut → canon/insights/docs propres).
- ✅ **Étape 4b (02-07)** : ingestion multi-format. `LUMINA-MEMORY-INGEST-TEXT` (`ugINEyyBKRLcDpH2`, executeWorkflowTrigger → chunk 3000/200 → embed → insert paramétré, table dérivée) = branche markdown/texte réutilisable. `AFTRSN-MEMORY-INGESTION/DRIVE-RECURSIVE` (`8h3AowpjR9IBaxYF`, Manual+trigger) : Drive Search (dossier) → filtre PDF+md/txt (docx exclus, décision Karter) → Download → Switch PDF/texte → Extract → Chunk → Embed → Insert `collection='canon'`. Testé bout en bout sur le dossier bible. Récursion 1 niveau (sous-dossiers profonds = amélioration future).
- ⬜ Boucle procédurale (scoring) ; timestamp `created_at` pour consolidation incrémentale (actuel : ORDER BY id DESC LIMIT 50) ; récursion Drive multi-niveaux

## Phase 2 — Agents AFTRSN
- ✅ Vague 1 câblée et publiée en PROD (5 agents + Channel-Content) — prompts v2 en anglais, structurés, validation Karter intégrée
- ✅ Outil Knowledge présent sur tous les agents
- ⬜ **Anomalies audit 02-07 (ordre de correction)** :
  - ✅ **A1 — CORRIGÉ 02-07** : les 3 tool-nodes obsolètes supprimés, 4 Call ajoutés (Culture-Steward, Experience-Designer, Secretary, Marketing) avec `$fromAI('query')`, connectés à l'AI Agent.
  - ✅ **A2 — CORRIGÉ 02-07** : prompt Secrétaire v2 collé, credential Postgres ajoutée au Chat Memory, workflow renommé **AFTRSN-Secretary** et **activé**.
  - ✅ **A3 — CORRIGÉ 02-07** : Hermes branché comme outil Ops. Sous-workflow `LUMINA-Hermes-Exec` (id `Dwv4rcMqNAyQzlrF`) : Login (credential Custom Auth `Hermes WebUI`, domaine restreint) → Code (cookie → session/new → chat/start → poll export). Outil `Call 'Hermes (Ops)'` ajouté sur Secrétaire + Marketing. Testé E2E : Secrétaire→Hermes→132 (44×3), Hermes seul→391 (17×23). Prérequis réglé : backend Hermes ré-authentifié (clés Coolify + redeploy).
  - ✅ **A4 — CORRIGÉ 02-07** : node Chat Memory Postgres ajouté à Experience-Designer (credential OK, connecté à l'AI Agent).
  - ✅ **A5 — CORRIGÉ 02-07** : prompt Channel-Content nettoyé — références Brand-Guardian/Acquisition-Performance retirées ; compliance de voix renvoyée au Marketing.
  - ✅ **A7 — CORRIGÉ 02-07** : les 6 nodes `lmChatOpenAi` (gpt-4.1-mini) remplacés par `lmChatOpenRouter` (credential OpenRouter account). Mapping doc v2 : Maestro+Marketing+Channel → `~anthropic/claude-sonnet-latest` ; Culture+Experience → `~google/gemini-flash-latest` ; Secrétaire → `~openai/gpt-mini-latest`. Testé : Maestro répond via claude-sonnet.
  - **→ Toutes les anomalies audit A1–A7 sont closes.**
- ⬜ Câblage mémoire à valider en exécution réelle (sessions, `$fromAI` pour Maestro)
- ✅ **Vague 2 (02-07)** : **AFTRSN-Comptable-Finance** (`F1wsXbsYZdEkwQ3n`) + **AFTRSN-Qualite-Compliance** (`zbW4p5qsBYhkfBdi`) créés (claude-sonnet, mémoire aftrsn, Knowledge), câblés comme outils sur Maestro (8 outils au total), prompt Maestro mis à jour. Testé : Maestro → Comptable → estimation budgétaire chiffrée. Content = déjà couvert par Channel-Content (vague 1). Boundaries : Comptable ne dépense/n'envoie jamais ; Qualité = challenger (verdict/risques/fixes), PII jamais au cloud.

## Phase 3 — Routage multi-LLM
- ✅ Stratégie + benchmark v2 (matrice `task_type`→LLM, arbre de décision, contrat JSON brand-agnostic)
- ✅ `LUMINA-AI-Router` bâti + publié (id `Cu8SozYmondKM8RB`) : Trigger → Code (mapping, alias `~…-latest`) → HTTP OpenRouter ; appel validé
- ✅ Garde-fou non négociable : `task_type='private'` (PII) → stop, jamais au cloud
- ✅ **A6 — CORRIGÉ 02-07** : Maestro → Router câblé (tool `Call 'LUMINA-AI-Router'`, contrat à plat task_type/instruction/context_raw + ligne ROUTING dans le prompt). Test bout en bout validé : `task_type research` → `google/gemini-3.5-flash-20260519`. Fix au passage : le trigger du Router (mode jsonExample) filtrait les champs entrants — exemple élargi aux deux contrats (plat + imbriqué).
- ✅ **Slugs vérifiés 02-07** : catalogue OpenRouter (338 modèles) — fallbacks pinnés existants (`claude-opus-4.8`, `gpt-5.5`, `gemini-3.5-flash`) ; **les alias `~…-latest` fonctionnent** (résolution constatée : `~openai/gpt-mini-latest`→`gpt-5.4-mini`, `~google/gemini-flash-latest`→`gemini-3.5-flash-20260519`). Mapping inchangé.
- ✅ **Injection RAG dans le Router (02-07)** : pour `copy/reasoning/analysis/draft`, le Code node appelle `memory-search` (brand) et injecte la mémoire pertinente dans `context_raw` avant l'appel LLM. Testé : tâche `copy` → mémoire de marque injectée + réponse.
- ⬜ Chaîne multi-LLM pour livrables critiques (prépa → créa → ciselage → validation)
- ⏸️ Modèle Hermes local gratuit (classify/extract/PII/LoRA) — GPU requis

## Phase 4 — Avancé / long terme
- ⬜ Auto-évaluation + matrice vivante (historique des perfs)
- ⬜ LoRA périodique Hermes (actif propriétaire)
- ✅ **LUMINA-PLAYBOOK v1 (02-07)** : tables mémoire généralisées (`<code>_memory` dérivé) + table `brand_profiles` + workflow `LUMINA-BRAND-PROVISION` (`Ul7u33t5VG8MDXWj`) + document blueprint. Validé : marque `demo` provisionnée, write+search OK sans edit de code.
- ✅ **Hermes apprenant (02-07)** : skills library + `skill_registry` + consultation/logging + graduation auto + auto-promotion nocturne
- ⬜ Durcissement sécurité (auth MCP, rotation secrets, gating humain)

## Transverse — Documentation & sauvegarde
- ✅ Pipeline `/ingest` + `/sync` ; wiki ai-automation (25 pages) : GitHub + projet Claude + Drive
- ✅ Règle CLAUDE.md : tout POS/marche à suivre/passation/milestone → projet Claude « Claude Lessons (1) » **ET** Drive
- ✅ **POS du 02-07 produits + double-sauvés** : câblage Maestro↔sub-agents+Router (exact+générique), audit/édition n8n par API interne (exact+générique), Hermes outil Ops (exact+générique) — + MILESTONE câblage + audit n8n live
- 🟡 **Rapatriement (02-07)** : ✅ 52 docs rapatriés + catégorisés dans le projet synchronisé (`archive-2026-06/` : POS-EXACT, POS-GENERIQUE, Marches-a-suivre, Passations-Milestones-Plans, Sessions-Architecture-Divers). ✅ Drive : 5 dossiers-catégories créés + POS-EXACT (5) + POS-GENERIQUE (3) + 1 Marche uploadés. ⏳ Reste ~40 fichiers Drive → **glisser-déposer** recommandé (les sous-dossiers de `archive-2026-06/` vers les dossiers Drive homonymes).
- ⬜ **Ménage destructif (à faire par Karter)** : `DROP TABLE IF EXISTS demo_memory; DELETE FROM memory_registry WHERE code='demo'; DELETE FROM brand_profiles WHERE code='demo';` + supprimer workflow `__tmp_hermes_reach_test` (garder/renommer `__tmp_skill_registry_ddl` → console SQL).
- 📌 Règle actée le 02-07 : **n8n fait foi sur la doc** en cas d'écart — vérifier les workflows avant tout travail dessus

---

## Déroulé d'exécution

**✅ Fait le 02-07 :**
1. ✅ **A1** — Tools de Maestro recâblés (4 sub-agents)
2. ✅ **A2** — Prompt Secrétaire corrigé + AFTRSN-Secretary activé
3. ✅ Slugs OpenRouter vérifiés — alias `~…-latest` validés en conditions réelles
4. ✅ **A6** — Maestro → Router câblé + test bout en bout réussi
5. ✅ **A3** — Hermes branché comme outil Ops (Secrétaire + Marketing), testé E2E ; backend Hermes ré-authentifié au passage
6. ✅ POS/marches à suivre du jour produits (exact + générique) + double sauvegarde

7. ✅ **A7** — Agents basculés sur OpenRouter par rôle (mapping doc v2), testé

8. ✅ **A4 / A5** — Chat memory ajoutée à Experience-Designer ; prompt Channel-Content nettoyé

9. ✅ **Mémoire épisodique + consolidation nocturne + RAG Router** (02-07)
10. ✅ **Hermes apprenant** : skills library (lumina+aftrsn) + `skill_registry` + consultation/logging + graduation auto (≥3 usages/≥90%, rétrograde au 1er échec) + auto-promotion nocturne (02-07)

11. ✅ **LUMINA-PLAYBOOK v1** — généralisation multi-marques (provisioning + brand_profiles + tables généralisées), validé marque démo (02-07)

12. ✅ Rapatriement documentaire (~30 POS/SOP anciens, catégorisés en archive-2026-06/) + étape 4b mémoire + **ménage** : marque `demo` supprimée, workflows `__tmp_*` traités
13. ✅ **Vague 2 des agents** — Comptable-Finance (`F1wsXbsYZdEkwQ3n`) + Qualité-Compliance (`zbW4p5qsBYhkfBdi`) créés, câblés sur Maestro (8 outils), testés (Maestro→Comptable→budget chiffré). Content = déjà couvert par Channel-Content (vague 1)
14. ✅ **PLAYBOOK v2** — (a) skills sur la banque de la marque active : Hermes-Exec « Build Skills Query » dynamique `lumina` + `<brand>_memory` (défaut aftrsn) ; (b) clonage du roster par script `cloneRoster(code,profile,{live})` (6 sous-agents rebindés + Maestro remappé, Router/Save Episode partagés préservés) — validé en dry-run ; POS double-sauvés

15. ✅ **Raffinements mémoire** (workflow `dVZyCwGYqnqMpS3P`, exec test 925 success) : (a) consolidation **incrémentale** — Get Episodes filtre `created_at >= NOW() - INTERVAL '26h'` ; (b) **dédup sémantique des insights** — insert `SELECT ... WHERE NOT EXISTS (embedding <=> vec < 0.12)` ; (c) **seuil de similarité skills** — Build Skills Query `AND (embedding <=> vec) < 0.35` (Hermes-Exec)

**⬜ Reste à faire (optionnel) :**
16. 📋 **Procédure prête** (à exécuter le jour d'une vraie 2ᵉ marque) : POS-GENERIQUE_Onboarding-nouvelle-marque_roster-clone_2026-07-02.md (projet Claude + Drive) — 8 étapes : provisioning → squelette dossiers+tag → canon → dry-run → `cloneRoster(live:true)` → rangement/tags → test délégation ×2 → clôture/rollback

---
*PLAN v3.1 — 2026-07-02, rédigé par Claude (Cowork). Ajout v3.1 : point 15 (consolidation incrémentale + dédup insights + seuil skills, testé exec 925). Roster AFTRSN complet ; boucle mémoire vivante affinée. Sources : docs recouvrés 29-06→01-07, passation 02-07, audit + câblage n8n live.*
