---
type: raw
title: "PASSATION_Knowledge-Architecture_2026-06-22-23"
source_url: "drive:1jFCpnMJFQHTeULFVboXu24zhbL61_Z4-"
captured: 2026-07-02
vault: brands
brand: AFTRSN
immutable: true
---

# PASSATION — Architecture de Connaissance & Mémoire Centrale
### À coller dans la conversation « Knowledge Architecture Guide — Hub Model »

**Période couverte :** ~2026-06-22 (20h) → 2026-06-23
**Pourquoi ce doc :** ce travail a été mené par erreur dans un autre chat ; il appartient à cette conversation. Ce MD restaure le contexte et l'état actuel pour continuer ici.
**Marque pilote / périmètre :** LUMINA (infra) · AFTER SUN PEOPLE / AFTRSN (marque pilote).

---

## 1. TL;DR (où on en est)

✅ **Fait :** le système de mémoire est **debout et validé de bout en bout**. La connaissance vit en markdown dans un repo Git (Obsidian), elle est ingérée dans **pgvector**, et les agents la récupèrent (via MCP / webhook).

⚠️ **Le point qui manque (prochaine priorité) :** la **vue humaine dans Notion** n'est **pas encore alimentée**. Obsidian/pgvector servent les *agents* ; il faut maintenant brancher **Obsidian → Notion** pour donner aux **humains** la connaissance utile, **par marque**. On commence par **AI Automation**.

---

## 2. Architecture de référence (modèle Git-hub)

> **Le repo Git est la source de vérité unique. pgvector et Notion sont des SORTIES dérivées, parallèles et régénérables — jamais en chaîne.**

```
[1] GitHub / Obsidian          [2] pgvector              [3] Agents / Claude
   (wiki = source de vérité) →  (mémoire = miroir wiki) →  (lisent la mémoire)
        ▲                            ▲                          │
   INTAKE (Drive→GitHub)        INGESTION (Wiki→pgvector)   MCP Memory Server
                                                            + Retrieval (Webhook)

   NOTION (vue humaine) ⟵ généré depuis le wiki  ← ⚠️ PAS ENCORE FAIT
```

- **Obsidian (vault = repo Git)** : `raw/` (sources curées immuables) + `wiki/` (pages propres, maintenues par le LLM) + `CLAUDE.md` (schéma) + `index.md` + `log.md`. Inspiré du « LLM Wiki » de Karpathy.
- **3 vaults** : `ai-automation` (existe), `brands` (à venir), `personal` (à venir). Pour l'instant seul **ai-automation** existe.
- **Détail complet** : voir `Architecture_Connaissance_Obsidian_Centric.md` (v2.1).

---

## 3. Ce qui a été fait (depuis ~20h hier)

### 3.1 Vault Obsidian
- Vault canonique unique : **`~/Documents/Lumina AI/GitHub/ai-automation`** (repo GitHub `frescatzi/ai-automation`, branche `main`).
- Monté dans VS Code par Karter (1ʳᵉ fois), synchronisé **Obsidian ↔ GitHub** via le plugin **Obsidian Git** (auto-commit/push).
- **Nettoyage** : supprimé un clone périmé (`~/Lumina/ai-automation`) et une maquette vide (`~/Documents/vaults/`). Gardé `~/Lumina/lumina-meta`. Un seul clone canonique reste.
- Structure interne : `wiki/architecture/`, `wiki/sop/`, `sessions/`, `raw/`, `CLAUDE.md`, `index.md`, `log.md`. Le wiki s'enrichit déjà de pages compilées par le LLM (`concept-*`, `synthese-*`).

### 3.2 Ingestion → pgvector (rebranchée Notion → GitHub)
- Ancienne ingestion = depuis **Notion** (obsolète). **Rebranchée pour lire le wiki GitHub.**
- Workflow `LUMINA — Memory — Ingestion (Wiki→pgvector)` :
  `Trigger → Truncate (asp_memory) → HTTP (GitHub Trees API recursive) → Split Out (tree) → Filter (wiki/*.md) → Get a file → Set Document → Chunk → Embed (OpenAI text-embedding-3-small) → Build Insert → Postgres Insert`
- **Idempotent** (TRUNCATE au début = reconstruction propre à chaque run).
- **Validé** : 13 fichiers wiki → ~35 chunks. Zéro reste Notion.
- 2 pièges réglés : Chunk faisait `$input.first()` → corrigé en `$input.all()` ; File Path doit être en **mode Expression**.

### 3.3 Récupération (validée)
- 3 portes vers pgvector : **MCP Server** (`search_brand_memory`, par où **Claude se connecte**), **Webhook** (HTTP), **Manual/Test**.
- Test de bout en bout OK : question → embedding → recherche → renvoie le bon extrait du wiki.

### 3.4 Organisation n8n (nommage, tags, dossiers, inventaire)
- **Convention :** `LUMINA — <Process> — <Détail>` pour l'infra ; `AFTRSN — …` pour le spécifique marque. Archives : préfixe `ZZZ —` + dossier `Z_ARCHIVES`.
- **Tags (1 par axe) :** Statut `🟢 prod`/`🟡 wip`/`🔵 test`/`⚫ deprecated` · Déclencheur `👆 manual`/`⏰ scheduled`/`🪝 webhook`/`⚡ event` · Domaine `🧠 memory`/`📥 intake`/`🤖 ai-agents`/`🔌 mcp`/`🛠️ util`.
- **Dossiers :** `AFTRSN/Agents` (Superviseur + Sub-Agents : Channel Content, Brand Guardian, Acquisition Performance) · `LUMINA-01-INTAKE` · `LUMINA-…-MEMORY` · `MCP` · `SandBox` · `Z_ARCHIVES`.
- **Obsolètes dépubliés** (important) : l'ancienne ingestion Notion (planifiée) a été **dépubliée** pour ne plus re-polluer pgvector.
- **Inventaire complet** : voir `Inventaire_Workflows_n8n_LUMINA.md`.

---

## 4. ⚠️ Le point manquant — Notion pour les humains

**Constat :** Notion n'est **pas alimenté**. Or il sert la **vue humaine** (lecture confortable, partage, mobile, par marque). Les agents, eux, sont servis (pgvector). Il manque le pont **Obsidian → Notion**.

**Objectif :** générer/synchroniser vers Notion le **sous-ensemble « à publier »** du wiki, **par marque**, en **sens unique** (Obsidian = source → Notion = vue dérivée, jamais éditée à la main).

**On commence par :** un **environnement de travail Notion pour AI Automation** (Karter le crée), puis un **test** de synchronisation Obsidian → Notion sur ce périmètre.

**Rappels de design :** Notion est **optionnel et non porteur** (le système tourne sans lui) ; c'est une **sortie dérivée** ; ne jamais en faire une source.

---

## 5. Prochaines étapes

1. **Brancher Obsidian → Notion** (vue humaine), périmètre **AI Automation** d'abord. Décider du mécanisme (workflow n8n `LUMINA — Publish — Notion`, ou autre) et du critère de publication (flag `publish: true` dans le frontmatter, ou dossier).
2. **Automatiser l'ingestion** (aujourd'hui manuelle) : webhook GitHub ou planifié, pour que pgvector se mette à jour à chaque push.
3. **Confirmer les déclencheurs des agents** (Superviseur/sous-agents).
4. **Publier le MCP Memory Server** s'il doit servir Claude en continu.
5. Plus tard : créer les vaults **`brands`** et **`personal`** quand ils auront du contenu ; exclure `wiki/README.md` de l'ingestion ; affiner la qualité de récupération.

---

## 6. Conventions & décisions clés (mémo)

- **Source de vérité = Git** ; pgvector + Notion = dérivés régénérables, jamais édités à la main.
- **Embedding figé** : `text-embedding-3-small` (1536 dim). Jamais le node Anthropic natif (bug 404).
- **Nommage** : LUMINA = infra ; AFTRSN = marque (les agents gardent AFTRSN).
- **Idempotence** : ingestion = TRUNCATE + reload (toujours un miroir propre).
- **Sécurité** : un workflow planifié+publié qui écrit dans pgvector doit être **dépublié** quand il devient obsolète.

---

## 7. Fichiers de référence produits (dans le dossier projet « Claude Lessons »)

- `Architecture_Connaissance_Obsidian_Centric.md` (v2.1) — **le design de référence**
- `Inventaire_Workflows_n8n_LUMINA.md` — tous les workflows + tags + dossiers
- `Brief_ClaudeCode_Montage_Vaults_Obsidian.md` — montage des vaults
- `Memoire_Centrale_ASP_Brief_Construction.md` · `Plan_Demarrage_Memoire_Centrale_MVP.md`
- `Claude_Review_Knowledge_Governance_Layer_v1.md` · `Instructions_Projet_ChatGPT_n8n_v2.md`
- SOPs : `SOP_installer-pgvector-sur-postgres-coolify.md`, `SOP_systeme-multi-agents-memoire-centrale-mcp-n8n.md`, `Guide-Connexion-Agents-AI-n8n.md`, `n8n-Brancher-API-et-Premier-Workflow.md`
- Sessions : `2026-06-19_session_memoire-centrale_etapes-1-2.md`, `2026-06-20_session_agents-orchestration-mcp.md`, `Session-Recap-n8n-Agents-AI.md`

---

## Version
v1.0 — Passation (session 2026-06-22/23) — à reprendre dans « Knowledge Architecture Guide — Hub Model »
