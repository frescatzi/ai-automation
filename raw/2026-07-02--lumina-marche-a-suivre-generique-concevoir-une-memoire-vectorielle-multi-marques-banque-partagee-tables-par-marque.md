---
type: raw
title: "LUMINA — Marche a suivre GENERIQUE — Concevoir une memoire vectorielle multi-marques (banque partagee + tables par marque)"
source_url: "drive:1mdpX3_AwMxOvlJUZXXvDXlgRw8DxmXm4"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre GÉNÉRIQUE — Concevoir une mémoire vectorielle multi-marques

**Portée :** patron réutilisable pour tout « AI-OS » gérant plusieurs marques/entités avec une mémoire vectorielle (pgvector ou équivalent) consommée par des agents.

> Explications en français ; identifiants techniques et SQL en anglais. Numérotation à partir de 1.

---

## 1. Principe directeur

Séparer **deux natures de connaissance** par une **frontière physique**, pas par un simple filtre :
- **Savoir réutilisable inter-entités** (process, solutions, patterns) → **une banque partagée**.
- **Savoir propre à une entité** (canon, identité, ops spécifiques) → **une banque par entité**.

Pourquoi physique (table/collection distincte) et pas un tag : un filtre oublié = fuite de contexte (l'agent d'une entité voit le contenu d'une autre, ou du système). La frontière dure supprime cette classe de bug par construction.

## 2. Granularité : quand choisir « une table par entité »

Choisir **une table (ou collection) par entité** quand au moins un de ces critères tient :
- horizon long (années) avec besoin de **clarté de support/maintenance** ;
- **volumes** appelés à devenir importants (perf de recherche + purge/backup par entité) ;
- le **stack entier** devient spécifique par entité (intégrations, plateformes, actifs distincts) → la mémoire doit suivre la même logique.

Sinon (peu d'entités, faible volume, infra à minimiser) : **une table + tag d'entité + filtrage** peut suffire — mais c'est un choix d'économie, pas de clarté.

## 3. Schéma type d'une banque

Colonnes minimales : `id`, `title`, `content`, `collection` (sous-catégorie), `knowledge_type`, `source`, `source_ref`, `content_hash UNIQUE` (upsert idempotent), `embedding VECTOR(d)`, `created_at`. Index : **HNSW cosine** sur `embedding` ; btree sur `collection`/`knowledge_type`.

Règle : si la frontière est **la table par entité**, **pas besoin de colonne d'entité** (l'entité = la table). Garder un schéma **identique** entre toutes les banques → migration et templates triviaux.

## 4. Sous-catégorisation interne

Frontières **dures** = la banque (entité). Frontières **douces** = des tags (`collection`, `knowledge_type`) à l'intérieur : ex. `canon`, `voice`, `ops`, `process`, `history`. Ça évite de multiplier les tables pour des sous-thèmes.

## 5. Convention de nommage

- Banque partagée : un nom stable (ex. `<infra>_memory`).
- Banque par entité : `<code-entité>_memory`, code lowercase court et stable.
- Outils/MCP : un outil **routé** `search_entity_memory(entity, question)` (un seul outil pour toutes les entités) + un outil pour la banque partagée.

## 6. Registre de routage (clé de la maintenabilité)

Une table de config `memory_registry(code, table_name, display_name, kind, active)` = **source unique de vérité**. Les workflows reçoivent un **code**, résolvent la **table** via le registre.

**Sécurité :** valider le nom de table contre le registre (**allowlist**) avant tout SQL. Ne jamais interpoler un nom de table arbitraire → injection.

## 7. Workflows templatisés (éviter N pipelines)

Un **modèle d'ingestion** et un **modèle de recherche** **paramétrés par la table** (résolue via registre). Ajouter une entité = paramétrer/cloner, pas reconstruire. Idéalement un seul workflow qui route selon `code`.

## 8. Câblage des agents

Chaque agent lit la **bonne banque** : agents propres à une entité → banque de cette entité ; agents transverses (dev/ops) → banque partagée ; certains croisent les deux (deux outils). La recherche reste un k-NN (`ORDER BY embedding <=> q LIMIT k`), éventuellement filtré par `collection` pour cibler une sous-catégorie.

## 9. Challenges & leçons

- **Mémoire mélangée + recherche sans filtre** → l'agent d'un domaine est noyé par le hors-sujet. *Leçon :* frontière physique entre natures de savoir.
- **Choix de granularité** → arbitrer selon l'**horizon opérationnel**, pas la simplicité du moment.
- **Table par entité = risque N pipelines** → templates paramétrés + registre + outil routé. *Leçon :* « ajouter une entité » doit être un clone paramétré.
- **Nom de table paramétré** → allowlist via registre (anti-injection).
- **Inserts** → requêtes paramétrées, jamais de concaténation. **Noms d'outils** → identiques entre node et prompts d'agents.
