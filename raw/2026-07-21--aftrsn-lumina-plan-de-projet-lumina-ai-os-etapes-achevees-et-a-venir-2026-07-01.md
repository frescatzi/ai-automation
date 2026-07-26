---
type: raw
title: "AFTRSN-LUMINA — Plan de projet LUMINA AI OS — etapes achevees et a venir — 2026-07-01"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — plan de projet lumina ai os — etapes achevees et a venir — 2026-07-01.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# Plan de projet — LUMINA AI OS / AFTER SUN PEOPLE (état au 2026-07-01)

**Vision :** plateforme multi-agents & multi-LLM sur n8n, brand-agnostic (1 workflow, N marques), avec mémoire vectorielle, routage LLM par tâche, et un bras d'exécution (Hermes). Marque pilote : **AFTER SUN PEOPLE (AFTRSN)**.

Légende : ✅ fait · 🟡 en cours / partiel · ⬜ à faire

---

## Phase 0 — Infrastructure
- ✅ n8n self-hosté (`n8n.aftersunpeople.com`, Hetzner/Coolify)
- ✅ PostgreSQL + **pgvector** (extension, HNSW cosine)
- ✅ Credentials modèles dans n8n (OpenAI, Claude, Gemini) + **OpenRouter account**
- ✅ **Hermes Agent** déployé sur Coolify, **https://hermes.aftersunpeople.com** (WebUI 8787, mot de passe)

## Phase 1 — Mémoire vectorielle (le socle)
- ✅ Pipeline d'ingestion GitHub→pgvector (chunk → embed OpenAI `text-embedding-3-small` → insert paramétré)
- ✅ Réparation pipeline (credential OpenAI morte, INSERT paramétré)
- ✅ Architecture **multi-banques** : `memory_registry` + `lumina_memory` (partagé) + `aftrsn_memory` (marque)
- ✅ Récupération par webhook + **MCP `search_brand_memory(brand, question)`** (routage par marque, option B)
- ✅ Rename `asp_memory` → `lumina_memory` ; récupération + ingestion repointées
- ✅ **Étape 4 (MVP) : bible de marque ingérée** dans `aftrsn_memory` (Drive PDF → Extract → embed → insert ; `collection='canon'`) ; Brand-Guardian la retrouve
- 🟡 Écriture mémoire **épisodique** (WRITE à chaque exécution) + **consolidation** nocturne — à câbler (§8 stratégie)
- ⬜ Étape 4b : récursion Drive (autres PDF) + branche markdown ; `.md` propres dans Obsidian `brands/aftrsn`
- ⬜ Filtrage `collection` dans la recherche ; boucle procédurale (scoring)

## Phase 2 — Agents AFTRSN (organisation)
- ✅ Socle hub-and-spoke bâti : Superviseur + Brand-Guardian + Channel-Content + Acquisition-Performance
- ✅ MCP mémoire relié aux agents (outil `Knowledge`)
- ✅ **Cadrage de l'organisation étendue** (Maestro + 3 experts + fonctions business + ops)
- ✅ Modèle de collaboration **décidé : orchestré + appels ciblés**
- ✅ Nom du Superviseur **décidé : Maestro** (`AFTRSN-Supervisor` → Maestro)
- ✅ Experts de domaine **validés** : Music & Sound Curator · Culture & Community Steward · Experience & Event Designer
- 🟡 Redéfinition complète des rôles — **gated sur Hermes (fait)** → à écrire
- ⬜ Écrire les system-prompts par rôle (identité, boundaries, langue, format, garde-fous, validation Karter)
- ⬜ Câbler la mémoire par agent : spécialistes/experts → `brand=aftrsn` ; Maestro → `lumina` + lecture `aftrsn`
- ⬜ 1ʳᵉ vague d'agents à construire (à trancher : socle assistant+curation d'abord)
- ⬜ Brancher Hermes comme outil Ops (web/fichiers/commandes) des agents

## Phase 3 — Routage multi-LLM (LUMINA AI OS)
- ✅ **Benchmark / stratégie de routage** livrée (matrice task_type → modèle, règles, contrat JSON)
- ✅ Réconciliation avec l'existant (pgvector déjà en prod ; clarif « 2 Hermes »)
- ✅ **AI Router MVP bâti + publié** (`LUMINA-AI-Router`) via **OpenRouter** (mapping task_type→slug + 1 appel HTTP) ; appel OpenRouter validé
- ✅ Garde-fou **PII/`private` → stop** (jamais au cloud)
- 🟡 Test du routage réel — à finir via un **caller** (Maestro appelle le router)
- ⬜ Vérifier/rafraîchir les slugs OpenRouter (churn) ; option `openrouter/auto`
- ⬜ Chaîne multi-LLM pour livrables critiques (Hermes prépa → GPT créa → Claude ciselage → Gemini validation)
- ⬜ Injecter la mémoire (RAG) dans `task_payload.context_raw` avant l'appel
- ⬜ (Différé) **modèle Hermes local gratuit** (classify/extract/PII/LoRA) — nécessite un **GPU**

## Phase 4 — Avancé / long terme
- ⬜ Auto-évaluation + validation croisée + historique des perfs → « matrice vivante »
- ⬜ Fine-tuning LoRA périodique d'Hermes (actif propriétaire)
- ⬜ Généralisation multi-marques (nouvelle marque = config JSON + `<code>_memory` + registre)
- ⬜ Durcissement sécurité (auth MCP, secrets, gating humain des actions critiques)

---

## Prochaines étapes immédiates (prochaine session)
1. Rafraîchir les slugs OpenRouter (`openrouter.ai/models`).
2. **Câbler Maestro → appelle `LUMINA-AI-Router`** (= vrai test de routage bout en bout).
3. Écrire les system-prompts par rôle + redéfinir les agents (Hermes est prêt).
4. Câbler la mémoire par agent (spécialistes→aftrsn ; Maestro→lumina+aftrsn).
