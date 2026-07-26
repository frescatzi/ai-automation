---
type: raw
title: "Inventaire_Workflows_n8n_LUMINA"
source_url: "drive:10aAATU4QftXNgrFpt2fgRfcoTQxRv1bX"
captured: 2026-07-02
vault: brands
brand: AFTRSN
immutable: true
---

# Inventaire des Workflows n8n — LUMINA

**Date :** 2026-06-23
**Périmètre :** tous les workflows n8n (`n8n.aftersunpeople.com`)
**But :** se repérer d'un coup d'œil — qui fait quoi, où c'est rangé, comment ça se déclenche, dans quel état.

---

## 1. Convention de nommage

> **`LUMINA — <Process> — <Détail>`** pour l'**infrastructure partagée**.
> **`AFTRSN — <Type> — <Détail>`** pour ce qui est **propre à la marque** After Sun People.

- **LUMINA** = parapluie / infra réutilisable (Intake, Memory, MCP, Util).
- **AFTRSN** = spécifique à la marque (ex. les agents qui pilotent ses opérations).
- Archives : préfixe **`ZZZ —`** (pour qu'elles tombent en bas) + dossier `Z_ARCHIVES`.

---

## 2. Système de tags (1 par axe → 3 max par workflow)

- **Statut :** `🟢 prod` · `🟡 wip` · `🔵 test` · `⚫ deprecated`
- **Déclencheur :** `👆 manual` · `⏰ scheduled` · `🪝 webhook` · `⚡ event`
- **Domaine :** `🧠 memory` · `📥 intake` · `🤖 ai-agents` · `🔌 mcp` · `🛠️ util`

Lecture : on voit instantanément **état + comment ça tourne + fonction**.

---

## 3. Structure des dossiers

```
📁 AFTRSN  (marque — After Sun People)
   📁 Agents
      AFTRSN — Agent — Supervisor
      📁 Sub-Agents
         AFTRSN — Agent — Channel Content
         AFTRSN — Agent — Brand Guardian
         AFTRSN — Agent — Acquisition Performance

— Infra partagée (LUMINA) —
📁 LUMINA-01-INTAKE        → Router (Drive→GitHub)
📁 LUMINA-02-MEMORY *(= "Collecting DATA", à renommer si tu veux)*
                          → Ingestion Wiki, Retrieval (manual + webhook), Console SQL
📁 MCP                     → Memory Server (search_brand_memory)
📁 SandBox (Test env.)     → essais
📁 Z_ARCHIVES              → obsolètes
```

---

## 4. Inventaire des workflows

### Infra — LUMINA

| Workflow | Rôle | Déclencheur | Dossier | Tags | Publié |
|---|---|---|---|---|---|
| **LUMINA — Memory — Ingestion (Wiki→pgvector)** | Lit le wiki GitHub → chunk → embed → remplit pgvector (TRUNCATE + reload, idempotent) | 👆 manuel | LUMINA-02-MEMORY | `🟢 prod` `👆 manual` `🧠 memory` | Non *(manuel)* |
| **LUMINA — Memory — Retrieval (Webhook)** | Endpoint HTTP : question → embed → recherche pgvector → réponse | 🪝 webhook | LUMINA-02-MEMORY | `🟢 prod` `🪝 webhook` `🧠 memory` | **Oui** |
| **LUMINA — Memory — Retrieval (Manual/Test)** | Même recherche, pour tester dans n8n | 👆 manuel | LUMINA-02-MEMORY | `🔵 test` `👆 manual` `🧠 memory` | Non |
| **LUMINA — Intake — Router (Drive→GitHub)** | Fichier Drive → Gemini trie → écrit dans le wiki GitHub (sinon rejeté) | ⚡ event (Drive) | LUMINA-01-INTAKE | `🟢 prod` `⚡ event` `📥 intake` | **Oui** |
| **LUMINA — MCP — Memory Server (search_brand_memory)** | Serveur MCP exposant l'outil `search_brand_memory` → par où Claude/LLM externes interrogent la mémoire | 🔌 MCP (server) | MCP | `🟢 prod` `🪝 webhook` `🔌 mcp` | À publier ⚠️ |
| **LUMINA — Util — Console SQL (Postgres)** | Bac à sable : lancer des requêtes SQL à la main (vérifs, TRUNCATE…) | 👆 manuel | LUMINA-02-MEMORY *(ou Util)* | `🛠️ util` `👆 manual` `🟢 prod` | Non |

### Marque — AFTRSN

| Workflow | Rôle | Déclencheur | Dossier | Tags | Publié |
|---|---|---|---|---|---|
| **AFTRSN — Agent — Supervisor** | Orchestrateur, point d'entrée du CEO, délègue aux spécialistes | à confirmer | AFTRSN / Agents | `🟢 prod` `🤖 ai-agents` | Oui |
| **AFTRSN — Agent — Channel Content** | Marketing, social, contenu & canaux | ~ appelé par Superviseur | AFTRSN / Agents / Sub-Agents | `🟢 prod` `🤖 ai-agents` | Oui |
| **AFTRSN — Agent — Brand Guardian** | Conformité & cohérence de marque | ~ appelé par Superviseur | AFTRSN / Agents / Sub-Agents | `🟢 prod` `🤖 ai-agents` | Oui |
| **AFTRSN — Agent — Acquisition Performance** | Acquisition, performance, KPIs | ~ appelé par Superviseur | AFTRSN / Agents / Sub-Agents | `🟢 prod` `🤖 ai-agents` | Oui |

### Archives — Z_ARCHIVES (obsolètes, à ne pas réactiver)

| Workflow | Pourquoi obsolète | Tags | Publié |
|---|---|---|---|
| **ZZZ — LUMINA — Notion Ingestion (deprecated)** | Ancienne ingestion Notion planifiée — remplacée par GitHub. ⚠️ **a été dépubliée** (sinon elle re-polluait pgvector) | `⚫ deprecated` `⏰ scheduled` `🧠 memory` | Non *(dépublié)* |
| **ZZZ — LUMINA — KB Single-file Ingestion (deprecated)** | Ancienne ingestion mono-fichier (collage manuel) — remplacée | `⚫ deprecated` `👆 manual` `🧠 memory` | Non |

---

## 5. Vue d'ensemble (le système)

```
[1] GitHub / Obsidian          [2] pgvector              [3] Agents / Claude
   (wiki = source vérité)   →   (mémoire, miroir wiki)  →  (lisent la mémoire)
        ▲                            ▲                          │
   Intake Router               Memory Ingestion            MCP Memory Server
   (Drive→GitHub)              (Wiki→pgvector)             + Retrieval (Webhook)
```

---

## 6. À finaliser (cases ouvertes)

- **Déclencheurs des agents** : confirmer comment Superviseur et sous-agents sont appelés (sous-workflow ? webhook ?).
- **Dossier des retrievals** : confirmer qu'ils vivent bien avec l'ingestion dans `LUMINA-02-MEMORY` (ou renommer « Collecting DATA »).
- **MCP Memory Server** : le publier s'il doit servir Claude en continu.
- **`LUMINA-Agents`** (ancien dossier) : à supprimer une fois les agents déplacés dans `AFTRSN/Agents`.
- **SandBox** : vérifier ce qu'il contient (workflow de test).

---

## Version
v1.0 — Inventaire workflows n8n LUMINA — 2026-06-23
