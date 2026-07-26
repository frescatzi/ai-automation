---
type: raw
title: "PASSATION_LUMINA-AI-OS_2026-07-02"
source_url: "drive:11Hi07fb5xEwfF8F9Opsz31nL1GXa75I_"
captured: 2026-07-06
vault: brands
brand: AFTRSN
immutable: true
---

# PASSATION — LUMINA AI OS / AFTER SUN PEOPLE

**Date de passation :** 2026-07-02
**Basée sur :** Plan de projet LUMINA AI OS (état au 2026-07-01) + POS/marches à suivre du 29-06 au 01-07
**Responsable projet :** Karter (Katel) — info@iamkarter.com — CEO AFTER SUN PEOPLE
**Statut global :** Socle complet et fonctionnel (infra + mémoire + router + Hermes) · agents 1ʳᵉ vague à câbler

---

## 1. Vision (rappel)

Plateforme **multi-agents & multi-LLM** orchestrée sur n8n, **brand-agnostic** (1 workflow, N marques), avec :
- une **mémoire vectorielle multi-banques** (pgvector),
- un **routage LLM par type de tâche** (AI Router via OpenRouter),
- un **bras d'exécution** (Hermes Agent),
- **Git = source de vérité** documentaire ; pgvector et Notion = dérivés régénérables.

Marque pilote : **AFTER SUN PEOPLE (AFTRSN)**. « After Sun » = l'événement (distinct de la marque).

---

## 2. État des lieux par phase (au 2026-07-02)

### Phase 0 — Infrastructure ✅ COMPLÈTE
- n8n self-hosté : `n8n.aftersunpeople.com` (Hetzner VPS + Coolify).
- PostgreSQL + pgvector (extension active, index HNSW cosinus).
- Credentials n8n : OpenAI, Anthropic, Gemini + **OpenRouter account**.
- **Hermes Agent** déployé sur Coolify : **https://hermes.aftersunpeople.com** (WebUI port 8787, mot de passe `SERVICE_PASSWORD_HERMESWEBUI`). POS : *POS EXACT — Hermes Agent sur Coolify — 2026-07-01*.

### Phase 1 — Mémoire vectorielle ✅ SOCLE FAIT
- Pipeline GitHub→pgvector opérationnel (chunk → embed `text-embedding-3-small` 1536 → INSERT paramétré).
- Réparé le 29-06 : credential OpenAI morte + passage à l'INSERT paramétré.
- Architecture **multi-banques** en place : `memory_registry` + `lumina_memory` (partagée, ex-`asp_memory` renommée le 30-06) + `aftrsn_memory` (marque).
- Récupération : webhook n8n + **MCP `search_brand_memory(brand, question)`** (routage par marque).
- **Bible de marque ingérée** dans `aftrsn_memory` (Drive PDF → embed, `collection='canon'`) — le Brand-Guardian la retrouve. (Étape 4 MVP faite le 30-06.)
- 🟡 Restant : mémoire épisodique (WRITE par exécution) + consolidation nocturne · récursion Drive (autres PDF) + branche markdown · filtrage `collection` dans la recherche.

### Phase 2 — Agents AFTRSN 🟡 EN COURS (priorité actuelle)
- Socle hub-and-spoke existant : Superviseur + Brand-Guardian + Channel-Content + Acquisition-Performance, MCP mémoire relié (outil `Knowledge`).
- **Décisions actées (30-06 / 01-07) :**
  - Modèle de collaboration : **orchestré + appels ciblés**. Karter = autorité finale sur toute action critique (dépense, envoi, publication, suppression).
  - Superviseur renommé **Maestro**.
  - **1ʳᵉ vague (5 agents)** : Maestro + Culture & Community Steward + Experience & Event Designer + Secrétaire + Marketing (events). Hermes = outil Ops.
  - **Music & Sound Curator retiré définitivement** (un LLM ne connaît pas les DJs de façon fiable).
  - 2ᵉ vague différée : Comptable & finance · Qualité & compliance · Content (Insta/newsletter/WhatsApp). Garde-voix de marque = Marketing.
- **System-prompts v2 rédigés** (doc *System-prompts 1ère vague v2 — 2026-07-01*) — **à valider par Karter puis câbler dans n8n**.
- Câblage mémoire défini : experts/spécialistes → `brand="aftrsn"` ; Maestro → `lumina` + lecture `aftrsn`.

### Phase 3 — Routage multi-LLM ✅ MVP BÂTI, TEST FINAL RESTANT
- Stratégie livrée (passation v2.0 + benchmark xlsx v2) : scoring des 4 moteurs, matrice `task_type` → LLM, arbre de décision, contrat JSON brand-agnostic.
- **Workflow `LUMINA-AI-Router` bâti + publié** (id `Cu8SozYmondKM8RB`) : Trigger (Accept all data) → Code (mapping `task_type` → slug OpenRouter, alias flottants `~…-latest` anti-churn) → HTTP Request OpenRouter (une clé pour Claude/GPT/Gemini). Appel OpenRouter validé.
- **Garde-fou non négociable** : `task_type='private'` (PII) → stop, jamais envoyé au cloud.
- Fallbacks pinnés si un alias casse : `anthropic/claude-opus-4.8`, `openai/gpt-5.5`, `google/gemini-3.5-flash`. Pas de Haiku.
- 🟡 Restant : test bout en bout via un **caller réel** (Maestro → Router) · injection RAG dans `task_payload.context_raw` · chaîne multi-LLM pour livrables critiques.
- ⏸️ Différé : modèle **Hermes local gratuit** (classify/extract/PII/LoRA) — nécessite un GPU non disponible.

### Phase 4 — Avancé / long terme ⬜
Auto-évaluation + matrice vivante · LoRA périodique Hermes · généralisation multi-marques (nouvelle marque = config JSON + `<code>_memory` + registre) · durcissement sécurité (auth MCP, rotation secrets, gating humain).

### Pipeline documentaire (transverse) ✅
- Slash commands `/ingest` et `/sync` (Claude Code) créés le 29-06 ; batch ingest de 12 sources ; archivage `raw/`→Drive (25 fichiers) ; purge manifeste durcie.
- Wiki `frescatzi/ai-automation/wiki/` (25 pages) répliqué le 01-07 dans le projet Claude « Claude Lessons » + sur le Drive (« Méthodes & SOP » + « Wiki ai-automation — Concepts & Architecture »).

---

## 3. Accès & identifiants clés

| Ressource | Valeur |
|---|---|
| n8n | `n8n.aftersunpeople.com` (Hetzner/Coolify) |
| Hermes WebUI | `https://hermes.aftersunpeople.com` (mdp : var Coolify `SERVICE_PASSWORD_HERMESWEBUI`) |
| Router n8n | workflow `LUMINA-AI-Router`, id `Cu8SozYmondKM8RB` |
| Repos GitHub | `frescatzi/ai-automation` · `frescatzi/brands` · `frescatzi/personal` |
| Tables mémoire | `memory_registry` · `lumina_memory` (partagée) · `aftrsn_memory` (marque) |
| Embedding (figé) | OpenAI `text-embedding-3-small` — 1536 dim, ne jamais changer sans tout ré-encoder |
| Notion DB | « AI Automation — Knowledge » `6fac8cb891e44c749e37a86419826905` (data_source `657827e3-…8687`, intégration AFTRN-n8n) |
| Drive | Lumina Inbox `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW` · Archive-Raw `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk` |

Secrets : jamais en clair — credentials n8n / variables Coolify uniquement.

---

## 4. Pièges & leçons figés (ne pas redécouvrir)

1. **Node Anthropic natif n8n = bug 404** → toujours HTTP Request pour Claude.
2. Coolify : **`SERVICE_FQDN_<SERVICE>_<PORT>`** crée la route Traefik (`SERVICE_URL_*` non) · domaine avec `https://` pour Let's Encrypt · un domaine = un service · **Redeploy** (pas Restart) après édition compose.
3. **Deux « Hermes »** : l'agent (exécution, API payantes) ≠ le modèle local gratuit (différé, GPU requis).
4. Slugs OpenRouter = **churn** → alias `~…-latest`, vérifier sur `openrouter.ai/models`, option `openrouter/auto`.
5. n8n : body JSON dynamique = **mode Expression + `JSON.stringify`** · node Code = une seule entrée · Run Once for Each Item + pairing `$('Node').item.json.x` · suppression toujours **après** écriture confirmée.
6. pgvector/Notion = dérivés : **jamais édités à la main**, régénérables depuis Git.

---

## 5. Prochaines étapes (dans l'ordre)

1. **Rafraîchir/vérifier les slugs OpenRouter** (`openrouter.ai/models`).
2. **Câbler Maestro → `LUMINA-AI-Router`** (Call n8n Workflow Tool) = vrai test de routage bout en bout.
3. **Valider les system-prompts v2** puis câbler les 5 agents de la 1ʳᵉ vague dans n8n.
4. **Câbler la mémoire par agent** : experts → `aftrsn` ; Maestro → `lumina` + `aftrsn`.
5. Brancher **Hermes comme outil Ops** des agents.
6. Ensuite : mémoire épisodique + consolidation nocturne · injection RAG dans le router · étape 4b (récursion Drive + markdown).

---

## 6. Références documentaires

- *Plan de projet LUMINA AI OS — étapes achevées et à venir — 2026-07-01* (dossier Claude Lessons)
- *LUMINA_AI_OS_Passation.md v2.0* + *Benchmark routage xlsx v2* (dossier Lumina OS)
- *POS EXACT — Hermes Agent sur Coolify — 2026-07-01*
- *POS EXACT — LUMINA-AI-Router (OpenRouter) — 2026-07-01*
- *System-prompts 1ère vague v2 — 2026-07-01*
- Marches à suivre EXACTES des 29-06 / 30-06 (multi-banques, réparation pipeline, rename + bible de marque)
- Wiki `ai-automation` (25 pages) : GitHub / projet Claude / Drive

---

*PASSATION v1.0 — rédigée le 2026-07-02 par Claude (Cowork), validée par Karter.*
