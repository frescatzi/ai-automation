---
type: wiki
title: Mémoire vectorielle multi-marques (banque partagée + tables par marque)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--lumina-marche-a-suivre-generique-concevoir-une-memoire-vectorielle-multi-marques-banque-partagee-tables-par-marque.md
  - raw/2026-07-02--lumina-ai-os-passation.md
  - raw/2026-07-02--lumina-playbook-v1-2026-07-02.md
  - raw/2026-07-21--aftrsn-lumina-marche-a-suivre-exacte-etape-1-architecture-memoire-multi-banques-schema-nommage-registre-2026-06-29.md
related:
  - wiki/concept-memoire-vivante-agents.md
  - wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
  - wiki/sop/sop-diagnostiquer-pipeline-memoire-vectorielle.md
  - wiki/sop/sop-clonage-roster-agents.md
  - wiki/synthese-lumina-ai-os.md
  - wiki/concept-pipeline-memoire-wiki-git.md
updated: 2026-07-21
---

# Mémoire vectorielle multi-marques (banque partagée + tables par marque)

## Origine de la décision

Décidé le 2026-06-29 (décideur : Katel/Karter) pour résoudre un problème constaté : la mémoire était **une seule table `asp_memory`** mélangeant contenu système/automation (dominant) et contenu de marque (rare), interrogée par un simple `ORDER BY embedding <=> vec LIMIT 5` **sans filtre** — le Gardien-marque était noyé dans les docs système et ne trouvait pas les faits de marque (ex. il ignorait que le genre est Afro House / Afro Tech / Melodic Afro House). D'où la frontière physique ci-dessous. `asp_memory` est devenue `lumina_memory` (rename/repurpose, pas une nouvelle table).

## Idée centrale

Séparer **deux natures de connaissance** par une **frontière physique**, pas par un simple filtre :
- **Savoir inter-entités** (process, solutions, patterns) → **banque partagée** (ex. `lumina_memory`).
- **Savoir propre à une entité** (canon, identité, ops) → **table par entité** (`<code>_memory`).

Un filtre oublié = fuite de contexte entre marques. La frontière physique supprime cette classe de bug par construction.

## Quand choisir « une table par marque »

Retenir ce pattern quand au moins un critère tient :
- **Horizon long** (années) — clarté de maintenance.
- **Volumes importants** — perf de recherche + purge/backup par entité indépendants.
- **Stack entier spécifique** par entité — la mémoire doit suivre la même logique.

Sinon (peu d'entités, faible volume) : une table + tag d'entité peut suffire — mais c'est un choix d'économie, pas de clarté.

## Schéma type d'une banque

Colonnes minimales :

```sql
id, title, content,
collection,        -- sous-catégorie : canon | episodic | insights | docs | skills
knowledge_type,    -- standard | procedural | ...
source, source_ref,
content_hash UNIQUE,  -- md5(content) → idempotence upsert
embedding VECTOR(d),
created_at
```

Index : **HNSW cosine** sur `embedding` ; btree sur `collection` / `knowledge_type`.

**Règle :** si la frontière est la table par entité, **pas besoin de colonne d'entité** (la table = l'entité). Schéma **identique** entre toutes les banques → migrations et templates triviaux.

## Sous-catégorisation interne (collections)

Frontières **dures** = la table (entité). Frontières **douces** = tags `collection` à l'intérieur :

| `collection` | Contenu |
|---|---|
| `canon` | Bible de marque, voix, personas, mots interdits (figé) |
| `episodic` | Traces d'exécution horodatées (brut) |
| `insights` | Connaissances consolidées (distillées du brut) |
| `docs` | Documents de référence ingérés |
| `skills` | Patterns appris, procédures validées |

Filtrer `collection` dans les recherches : exclure par défaut `episodic` pour ne pas polluer les recherches de référence.

> **Évolution de la taxonomie :** la spec figée du 2026-06-29 proposait un starter différent — `canon | voice | ops | process | history` côté marque, `automation | ai_os | pattern` côté `lumina_memory`. La table ci-dessus (`canon | episodic | insights | docs | skills`) est la version qui a suivi (2026-07-02) et qui fait foi aujourd'hui. Les deux partagent le même principe (frontière dure = table, frontière douce = `collection`).

## Registre de routage (`memory_registry`)

Table de config = source unique de vérité :

```sql
memory_registry(code, table_name, display_name, kind, active)
```

Les workflows reçoivent un **code** → résolvent la **table** via le registre. **Sécurité :** valider le nom de table contre le registre (allowlist) avant tout SQL. Ne jamais interpoler un nom de table arbitraire → risque d'injection SQL.

## Mapping amont (coffre Obsidian) → banque pgvector (aval)

La séparation par entité existe déjà **en amont**, au niveau des coffres/dossiers Obsidian ; l'architecture ci-dessus l'aligne **en aval**, au niveau des tables :

| Coffre / vault Obsidian (amont) | → `wiki/` → ingestion → | Banque pgvector (aval) |
|---|---|---|
| `ai-automation` (IA, API, MCP, agents, automation) | pipeline | `lumina_memory` |
| `brands` (1 **dossier par marque**) | pipeline | **1 table par marque** (`<code>_memory`, ex. `aftrsn_memory`) |
| `personal` (perso/business/général) | — | **non ingéré** (les agents n'y touchent pas) |

Le **dossier-par-marque** en amont mappe proprement sur la **table-par-marque** en aval. Point ouvert (à confirmer) : que les **codes de dossier de marque** dans le coffre `brands` correspondent bien aux **codes du `memory_registry`** → routage cohérent de bout en bout (dossier → `wiki/` → table). Voir [[concept-pipeline-memoire-wiki-git]] pour le détail du pipeline `wiki/` → base vectorielle.

## Workflows paramétrés (éviter N pipelines)

Un **modèle d'ingestion** et un **modèle de recherche** paramétrés par la table résolue via registre. Ajouter une entité = paramétrer/cloner, pas reconstruire. Idéalement 1 workflow unique qui route selon `code`.

## Câblage des agents

- Agents propres à une entité → banque de cette entité.
- Agents transverses (dev/ops, skills) → banque partagée.
- Certains croisent les deux (deux outils `search`).

Recherche : k-NN `ORDER BY embedding <=> q LIMIT k`, éventuellement filtré par `collection`.

## Conventions de nommage

- Banque partagée : `<infra>_memory` (ex. `lumina_memory`).
- Banque par entité : `<code>_memory` (code lowercase court et stable, ex. AFTRSN → `aftrsn`).
- Workflows infra (templates partagés) : préfixe `LUMINA-` (ex. `LUMINA-MEMORY-INGESTION/...`).
- Outil MCP/n8n : la spec figée du 2026-06-29 distinguait 2 outils — `search_brand_memory(brand, question)` (routé par marque) et `search_process_memory(question)` → toujours `lumina_memory`. En pratique, viser 1 outil routé `search_entity_memory(entity, question)` pour toutes les entités (marque ou partagé) plutôt que deux noms distincts. **Le nom retenu doit être identique entre le node n8n et les prompts des agents.**

## Pièges & leçons

- **Mémoire mélangée sans filtre** → l'agent d'une marque est noyé par le hors-sujet. Frontière physique d'abord.
- **Table par entité = risque N pipelines** → templates paramétrés + registre + outil routé.
- **Nom de table paramétré** → allowlist via registre, jamais d'interpolation directe.
- **Inserts** → requêtes paramétrées (`$1..$n`), jamais de concaténation.
- **Noms d'outils** → identiques entre node n8n et prompts des agents.

## Voir aussi

- [[concept-memoire-vivante-agents]] — le cycle WRITE/READ/CONSOLIDATE sur cette banque.
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — installation pgvector, index HNSW.
- [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] — réparer un pipeline mémoire cassé.
- [[sop/sop-clonage-roster-agents]] — provisionner une nouvelle banque marque + cloner les agents.
- [[synthese-lumina-ai-os]] — le contexte système Lumina.
- [[concept-pipeline-memoire-wiki-git]] — le pipeline `wiki/` (Git) → cette base vectorielle (Partie 2), qui alimente le mapping coffre → banque ci-dessus.
