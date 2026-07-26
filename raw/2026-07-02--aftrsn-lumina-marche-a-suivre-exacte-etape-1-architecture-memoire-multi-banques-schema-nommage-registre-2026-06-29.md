---
type: raw
title: "AFTRSN-LUMINA — Marche a suivre EXACTE — Etape 1 architecture memoire multi-banques (schema + nommage + registre) — 2026-06-29"
source_url: "drive:1ExDhxZbP0Sfn41nUrC4EGK0D8ZWknSPr"
captured: 2026-07-02
vault: brands
brand: aftrsn
immutable: true
---

# Marche à suivre EXACTE — Étape 1 : architecture mémoire multi-banques (schéma + nommage + registre)

**Date :** 2026-06-29
**Décideur :** Katel (Karter)
**Objet :** figer la fondation de la mémoire vectorielle multi-marques avant de construire.

> Convention : explications en français ; noms de tables/colonnes/champs et SQL en anglais. Numérotation à partir de 1.

---

## 1. Pourquoi ce chantier (le problème constaté)

La mémoire actuelle est **une seule table `asp_memory`** qui mélange :
- du contenu **système/automation** (wiki LUMINA : architecture, n8n, mémoire) — qui domine,
- très peu de contenu **de marque**.

Et la recherche fait `ORDER BY embedding <=> vec LIMIT 5` **sans aucun filtre**. Conséquence vécue : le Gardien-marque, en interrogeant la mémoire, est **noyé dans les docs système** et ne trouve pas les faits de marque (ex. il n'a pas su que le genre est Afro House / Afro Tech / Melodic Afro House).

**Décision :** séparer la mémoire en banques distinctes.

## 2. Décision d'architecture (figée)

- **`lumina_memory`** — UNE table partagée : connaissance des **process & solutions réutilisables d'une marque à l'autre** (automation/infra). Le contenu actuel de `asp_memory` est en réalité ça → il devient `lumina_memory`.
- **`<code-marque>_memory`** — **UNE table par marque** : canon + connaissance/ops propres à la marque. Ex. AFTER SUN PEOPLE → **`aftrsn_memory`**.

**Pourquoi une table par marque (et pas un simple tag) :** clarté de support et de maintenance sur 6 mois → 3 ans ; volumes qui deviendront colossaux ; et surtout des **process/plateformes brand-spécifiques** à venir (Wix, Hermes, etc.) gérant des actifs distincts par marque → l'isolation par marque doit être cohérente sur **tout le stack**, pas que la mémoire.

**Frontières :** dure = la **marque** (= la table). Douce = des **tags** à l'intérieur de chaque table (colonnes `collection` / `knowledge_type`).

## 3. Schéma figé (identique pour `lumina_memory` et chaque `<marque>_memory`)

```sql
CREATE TABLE <table_name> (
  id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title         TEXT,
  content       TEXT NOT NULL,
  collection    TEXT,           -- bucket de sous-catégorie (voir taxonomie §5)
  knowledge_type TEXT,          -- type plus fin (optionnel)
  source        TEXT,           -- système d'origine: 'GitHub' | 'Notion' | ...
  source_ref    TEXT,           -- chemin/id d'origine: 'wiki/brand/genres.md'
  content_hash  TEXT UNIQUE,    -- md5(content) -> upsert idempotent
  embedding     VECTOR(1536),   -- OpenAI text-embedding-3-small
  created_at    TIMESTAMPTZ DEFAULT now(),
  updated_at    TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON <table_name> USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON <table_name> (collection);
CREATE INDEX ON <table_name> (knowledge_type);
```

Notes :
- **Pas de colonne `brand`** : la marque est implicite dans le **nom de la table** (table-per-brand). Idem `lumina_memory` (partagée).
- `content_hash UNIQUE` permet `INSERT ... ON CONFLICT (content_hash) DO NOTHING` (idempotence, déjà en place).
- On garde le schéma actuel éprouvé (mêmes colonnes que `asp_memory`) → migration triviale.

## 4. Convention de nommage figée

| Élément | Règle | Exemple |
|---|---|---|
| Banque automation | `lumina_memory` | `lumina_memory` |
| Banque de marque | `<code-marque>_memory` (code lowercase, court, stable) | `aftrsn_memory` |
| Code marque | version lowercase du préfixe de marque | AFTRSN → `aftrsn` |
| Workflows infra | préfixe `LUMINA-` (templates partagés) | `LUMINA-MEMORY-INGESTION/...` |
| Outil MCP marque | `search_brand_memory(brand, question)` (routé) | — |
| Outil MCP process | `search_process_memory(question)` → `lumina_memory` | — |

## 5. Taxonomie figée (tags internes `collection`)

Starter (peut évoluer) :
- `canon` — bible / identité de marque (nom, positionnement, genres, palette…)
- `voice` — ton, vocabulaire, do/don't
- `ops` — opérations propres à la marque
- `process` — workflows brand-spécifiques
- `history` — décisions/événements passés

(`lumina_memory` utilisera p. ex. `collection = 'automation' | 'ai_os' | 'pattern'`.)

## 6. Registre figé (routage data-driven + sécurité)

Une table de config, **source unique de vérité** pour le routage :

```sql
CREATE TABLE memory_registry (
  code         TEXT PRIMARY KEY,   -- 'aftrsn', 'lumina', ...
  table_name   TEXT NOT NULL,      -- 'aftrsn_memory', 'lumina_memory'
  display_name TEXT,               -- 'AFTER SUN PEOPLE'
  kind         TEXT NOT NULL,      -- 'brand' | 'shared'
  active       BOOLEAN DEFAULT true,
  created_at   TIMESTAMPTZ DEFAULT now()
);
-- seed
INSERT INTO memory_registry (code, table_name, display_name, kind) VALUES
  ('lumina','lumina_memory','Automation & Process (shared)','shared'),
  ('aftrsn','aftrsn_memory','AFTER SUN PEOPLE','brand');
```

**Règle de sécurité (importante) :** les workflows reçoivent le **code** de marque, puis **résolvent le nom de table via `memory_registry` (allowlist)**. On n'interpole **jamais** un nom de table arbitraire venu de l'extérieur dans le SQL → sinon injection. Seuls les noms validés par le registre sont utilisés.

## 6bis. Source amont : Obsidian / Git (INCHANGÉ)

Rappel de l'archi « Git-hub » (modèle Karpathy, décidée le 2026-06-20) : la **source de vérité = le repo Git** (vault ouvert dans Obsidian, couche `wiki/` écrite par le LLM). **pgvector et Notion sont des sorties DÉRIVÉES en parallèle** du Git — pas une chaîne via Notion. **Les agents lisent pgvector** (jamais Notion ni le repo directement). Donc **rien ne change côté Obsidian** : la séparation par coffre y est déjà faite, on aligne juste l'aval (les tables) dessus.

Mapping coffre (amont) → banque (aval) :

| Coffre / vault Obsidian (amont) | → `wiki/` → ingestion → | Banque pgvector (aval) |
|---|---|---|
| **`ai-automation`** (IA, API, MCP, agents, automation) | pipeline | `lumina_memory` |
| **`brands`** (1 **dossier par marque** dans ce coffre) | pipeline | **1 table par marque** : `<code>_memory` (ex. `aftrsn_memory`) |
| **`personal`** (perso/business/général) | — | **non ingéré** (les agents n'y touchent pas) |

Nuance importante : la séparation des marques en amont est **au niveau dossier** dans le coffre `brands` ; en aval, elle devient **une table par marque**. C'est l'évolution actée aujourd'hui (avant, l'isolation marque était seulement folder/tag dans un store unique). Le **dossier-par-marque** amont mappe donc proprement sur la **table-par-marque** aval.

À confirmer en étape 2 : que les **codes de dossier de marque** (coffre `brands`) correspondent aux **codes du registre** `memory_registry` → routage cohérent de bout en bout (dossier → `wiki/` → table).

## 7. Comment rester maintenable (le point clé)

Pour qu'« une table par marque » ne devienne pas N pipelines à la main :
1. **Workflows templatisés/paramétrés** : un modèle d'ingestion et un modèle de recherche qui prennent le **code marque → table** en paramètre (résolu via le registre). Ajouter une marque = paramétrer, pas reconstruire.
2. **Registre** comme aiguilleur unique.
3. **MCP routé par marque** : un seul outil `search_brand_memory(brand, question)` au lieu d'un par marque.

## 8. Étapes suivantes (après cette spec)

2. Renommer/repurposer `asp_memory` → `lumina_memory`.
3. Créer `aftrsn_memory` (schéma §3) + `memory_registry` (§6).
4. Templatiser ingestion + recherche (paramètre = table via registre).
5. Ingérer la bible dans `aftrsn_memory` (`collection='canon'`).
6. Câbler Gardien-marque → `aftrsn_memory` ; agents ops → `lumina_memory` ; MCP routé.
7. Re-test bout-en-bout (le « techno vs Afro House » doit enfin réussir).

## 9. Challenges, solutions, leçons

- **Challenge :** mémoire unique mélangée (système + marque) + recherche sans filtre → le Gardien-marque ne trouvait pas les faits de marque.
  **Solution :** séparer en banques ; frontière dure par marque (table).
  **Leçon :** séparer « comment on opère » de « ce qu'est la marque » par une **frontière physique**, pas par un filtre qu'on peut oublier.
- **Challenge :** choisir la granularité (tag vs table unique vs table par marque).
  **Solution :** table par marque, vu l'horizon (3 ans), les volumes, et un **stack** qui devient brand-spécifique (Wix, Hermes…).
  **Leçon :** dimensionner la topologie mémoire pour l'**horizon opérationnel**, pas pour la simplicité MVP du moment.
- **Challenge/risque :** table par marque → N pipelines à maintenir.
  **Solution :** templates paramétrés + registre + MCP routé.
  **Leçon :** rendre « ajouter une marque » = un **clone paramétré**, jamais une reconstruction.
- **Leçon sécurité :** paramétrer un nom de table impose une **allowlist via le registre** (jamais d'interpolation libre → injection SQL).
- **Leçon héritée (session précédente) :** inserts en **requêtes paramétrées** (pas de concaténation) ; le nom de l'outil mémoire doit être **identique** entre le node et les prompts des agents.
