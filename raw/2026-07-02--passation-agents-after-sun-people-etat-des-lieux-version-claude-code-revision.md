---
type: raw
title: "PASSATION_Agents-AFTER-SUN-PEOPLE_etat-des-lieux_version-Claude-Code_revision"
source_url: "drive:1VDe4fYQyCMVCXhbmAx_x65sp-4od29Cf"
captured: 2026-07-02
vault: brands
brand: AFTRSN
immutable: true
---

# Passation — Agents AFTER SUN PEOPLE : état des lieux, version Claude Code & révision

> **But** : ouvrir une session dédiée aux **agents**. Trois objectifs : (1) **état des lieux** précis des 4 agents existants (n8n + MCP), (2) **porter/compléter en version Claude Code** (`.claude/agents/`), (3) **réviser** les agents ensemble (system prompts, périmètres, outils, garde-fous).
> **Date** : 2026-06-29. **Fondateur / validateur final** : Katel Tindjou (= Karter), partners@aftersunpeople.com.
> ⚠️ **À revérifier en début de session** : l'état décrit ci-dessous vient de notes du ~2026-06-20. Avant de t'appuyer dessus, ouvre n8n et confirme que les 4 workflows agents existent toujours et tournent.

---

## 1. Ce qu'on construit / révise (en une phrase)

Un système **multi-agents hub-and-spoke** pour les opérations d'AFTER SUN PEOPLE : **1 Superviseur** qui orchestre **3 spécialistes**, chacun lisant la **mémoire centrale** (bible de marque) avant d'agir, avec **validation finale humaine** (Katel) sur le critique. Il existe déjà en **n8n** ; on veut en **état des lieux**, une **version Claude Code**, et une **révision**.

---

## 2. État des lieux — ce qui est DÉJÀ fait (n8n)

**Architecture : 4 agents = 1 Superviseur + 3 spécialistes (hub-and-spoke).** Source de vérité du design : page Notion « Brand Source of Truth » (sous-page de « The Backstage »).

| Agent (préfixe `AFTRSN`) | Rôle | Skills associés |
|---|---|---|
| **AFTRSN-Supervisor** | Orchestre, point d'entrée de Katel, médie entre spécialistes | — |
| **AFTRSN-Brand-Guardian** | Cohérence & voix de marque, conformité | `brand-review` |
| **AFTRSN-Channel-Content** | Contenu & canaux (IG, newsletter, site, calendrier Canva) | `draft-content`, `email-sequence`, `canva-creator` |
| **AFTRSN-Acquisition-Performance** | Campagnes, KPIs, acquisition, recaps post-édition | `campaign-plan`, `performance-report` |

**Ce qui fonctionne (testé, ~2026-06-20) :**
- **Superviseur** : Chat Trigger → AI Agent (OpenAI **gpt-4o-mini**) + outil **`Knowledge`** (HTTP → webhook `memory-search`) + **`Chat memory`** (Postgres Chat Memory, table `n8n_chat_histories`, fenêtre 15). Teste OK : appelle la bible + garde le fil. **Recette réutilisable** (dupliquée pour les 3 spécialistes).
- **3 spécialistes** : montés par duplication du Superviseur, system message propre (en anglais) : identité + règle de nommage AFTRSN + « always query Knowledge before answering » + boundaries (rester dans son domaine, déférer aux autres) + « Katel valide le critique » + « Respond in the user's language ».
- **Orchestration** : le Superviseur appelle chaque spécialiste via un **« Call n8n Workflow Tool »** (query = `$fromAI`). Chaque spécialiste a un trigger **« When Executed by Another Workflow »** (champ `query`), source prompt = `{{ $json.query || $json.chatInput }}`.
- **Serveur MCP** : `AFTRSN-MCP-Server` = **MCP Server Trigger** + outil **`search_brand_memory`** (POST `https://n8n.aftersunpeople.com/webhook/memory-search`). Connecté dans Claude (Customize > Connectors, SSE, sans auth pour le MVP) et **testé depuis Claude/Cowork** → renvoie les entrées de la bible. La mémoire centrale est donc accessible depuis **n8n ET Claude**.

**Runtime actuel des agents = n8n** (choisi pour l'autonomie). Tokens = **OpenAI**, pas Claude.

---

## 3. Pièges déjà résolus (à ne pas refaire)

- **Chat memory d'un spécialiste plante quand il est appelé en sous-workflow** (pas de session chat) → fix : Session ID « Define below » = `{{ $json.sessionId || 'subagent' }}` (alternative : retirer la Chat memory des spécialistes, seul le Superviseur garde la mémoire conversationnelle).
- **Les spécialistes doivent être ACTIFS** (Published) pour être appelables par le Superviseur.
- **Node natif Anthropic n8n = bug 404** → utiliser HTTP Request pour Claude (et de toute façon, embeddings = OpenAI `text-embedding-3-small`).
- **Nommage** : l'outil mémoire se nomme **`Knowledge`** (le system prompt doit le référencer par ce nom) ; l'historique = **`Chat memory`**.

---

## 4. Intégration dans le système Lumina

- **Mémoire centrale** : PostgreSQL + pgvector (`asp_memory`, 1536, HNSW cosine) sur le VPS Hetzner/Coolify. Alimentée depuis le **wiki GitHub** (pipeline Lumina, **complet et automatisé** : Drive → GitHub → pgvector + Notion).
- **3 portes d'accès à la mémoire** : Manuel/Test, **Webhook** (`memory-search`, la porte que les agents appellent en prod), et **MCP Server** (`search_brand_memory`, pour les clients LLM comme Claude).
- **Bible de marque** = « Brand Source of Truth » (Notion) = la référence que **chaque agent lit avant d'agir**. ⚠️ **Prérequis qualité** : tant que la bible n'est pas suffisamment riche, le Brand-Guardian aura peu de matière. À évaluer.

---

## 5. Objectif A — Version Claude Code (`.claude/agents/`)

**Pourquoi** : avoir les agents aussi en **Claude Code** (en plus de n8n) pour **piloter le repo/vault depuis Claude Code** (sur Claude, pas OpenAI), avec délégation native du Superviseur aux sous-agents.

**Comment ça marche (Claude Code)** : des sous-agents = fichiers **`.claude/agents/<nom>.md`** avec frontmatter (`name`, `description`, `tools`, `model`) + un **system prompt**. Le Superviseur (l'agent principal) **délègue** à un sous-agent via l'outil de tâches. Permissions d'outils à définir par agent.

**À décider au build** :
- **Dans quel repo** vivent les agents Claude Code ? (`aftersunpeople-ops` semble le bon foyer brand-spécifique ; sinon `ai-automation`.)
- **Parité ou divergence** avec la version n8n (mêmes 4 agents, mêmes périmètres ? quels outils/skills côté Claude Code ?).
- **Connexion mémoire** : les sous-agents interrogent `search_brand_memory` (MCP) — confirmer l'accès depuis Claude Code.
- **Répartition des rôles n8n vs Claude Code** : n8n = autonomie/prod (déclencheurs, planifié) ; Claude Code = pilotage repo, dev, tâches interactives. Clarifier qui fait quoi.

---

## 6. Objectif B — Révision des agents (à faire ensemble)

Checklist de revue, pour les 4 agents :
- **System prompts** : identité, périmètre (boundaries), règle de déférence inter-agents, « query Knowledge first », langue de réponse, garde-fou validation Katel.
- **Alignement avec la Brand Source of Truth** maintenant que le pipeline mémoire est complet (la bible est-elle à jour / assez fournie ?).
- **Outils/skills** réellement branchés vs déclarés (Brand-Guardian → `brand-review` ; Channel Content → `draft-content`/`email-sequence`/`canva-creator` ; Acquisition Performance → `campaign-plan`/`performance-report`).
- **Boucle de correction** : les agents re-corrigent les éléments flaggés jusqu'à approbation de Katel — vérifier qu'elle est réellement implémentée.
- **Déclencheurs réels** : aujourd'hui les agents tournent « à vide » → définir leurs entrées/sorties concrètes (quelles tâches, quels événements).
- **Durcissement** : auth MCP (**OAuth2** au lieu de « sans auth ») ; marquer Katel-gated les tâches critiques (coût, sécurité, accès API, suppression mémoire).

---

## 7. Questions ouvertes à trancher en début de session

1. **Repo** des agents Claude Code (`aftersunpeople-ops` vs `ai-automation`).
2. **Parité n8n ↔ Claude Code** : on duplique les 4 mêmes agents, ou la version Claude Code a un périmètre différent ?
3. **Modèle** : n8n = OpenAI gpt-4o-mini ; Claude Code = Claude. OK de diverger ?
4. **Bible de marque** : assez complète pour rendre le Brand-Guardian utile, ou bloc prioritaire avant la révision ?
5. **Répartition d'usage** n8n (prod/autonome) vs Claude Code (pilotage/dev).

---

## 8. Pré-requis avant le build

- [ ] Confirmer en direct dans n8n que les 4 agents existent et tournent (notes datées).
- [ ] Vérifier l'accès à `search_brand_memory` (MCP) depuis Claude Code.
- [ ] Identifier/ouvrir le repo cible des agents Claude Code.
- [ ] Évaluer la richesse actuelle de la Brand Source of Truth (Notion).

---

## 9. Infos techniques (sans secrets)

- **n8n** self-hosté : `n8n.aftersunpeople.com` (Hetzner/Coolify), v2.10.4.
- **Agents n8n** : préfixe `AFTRSN` (brand-spécifique), dossier brand `AFTRSN` (Supervisor + sous-dossier `Sub-Agents/`). Outil mémoire = **`Knowledge`** (HTTP → webhook `memory-search`) ; historique = **`Chat memory`** (Postgres).
- **MCP** : `AFTRSN-MCP-Server`, outil `search_brand_memory` (POST webhook `memory-search`, SSE en prod). Auth = aucune (MVP) → à durcir.
- **Mémoire** : pgvector `asp_memory` (1536, HNSW), embeddings OpenAI `text-embedding-3-small`. Notion « Brand Source of Truth » (sous « The Backstage »).
- **Conventions** : contenu des champs d'agent (prompts, noms de nodes, descriptions d'outils) **en anglais** ; explications en français. Préfixe `LUMINA` = infra partagée ; `AFTRSN` = items brand (dont les agents).
- **Sécurité** : aucun token/PAT en conversation — secrets dans n8n.

---

## 10. Prompt d'ouverture pour la nouvelle conversation

> On travaille sur les **agents AFTER SUN PEOPLE**. Lis cette passation. Commence par **(1) un état des lieux vérifié en direct** (ouvre n8n, confirme que les 4 agents — Supervisor, Brand-Guardian, Channel-Content, Acquisition-Performance — existent et tournent, et que le MCP `search_brand_memory` répond). Ensuite **fais-moi trancher les 5 questions ouvertes**. Puis on attaque la **version Claude Code (`.claude/agents/`)** et la **révision des agents** ensemble, étape par étape — je suis pragmatique et j'apprends en faisant, donc explications concrètes et un pas à la fois.

---

## 11. À sauvegarder automatiquement (en fin de session)

Comme pour les autres chantiers, produire à la fin :
- un **.md « marche à suivre exacte »** (le build/révision réel, pas à pas) ;
- un **.md « marche à suivre générique »** (pattern réutilisable multi-agents) ;
- un **POS exact** et un **POS générique**.

La marche à suivre et les POS doivent inclure : les **challenges rencontrés + solutions + leçons apprises**, une **explication de chaque agent** (rôle, périmètre, outils, system prompt) et de l'orchestration — qui fait quoi, quand, où, comment et pourquoi. Titres de fichiers **précis et auto-explicites**.
