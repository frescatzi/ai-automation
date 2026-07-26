---
type: raw
title: "AFTRSN-LUMINA — POS EXACT — Runbook memoire multi-banques (ajouter une marque, ingerer, router)"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — pos exact — runbook memoire multi-banques (ajouter une marque, ingerer, router).md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# POS EXACT — Runbook : mémoire multi-banques AFTRSN / LUMINA

**Procédure Opérationnelle Standardisée (spécifique à ce système).**
**Cible :** pgvector sur Postgres (Hetzner/Coolify) · n8n `n8n.aftersunpeople.com` · MCP `LUMINA-MCP-Server`.

> Référence d'exploitation. Détail pédagogique : voir la « Marche à suivre EXACTE — Étape 1 ».

---

## 1. Cartographie cible (référence)

- **`lumina_memory`** — banque partagée (automation/process). = l'ancien `asp_memory` repurposé.
- **`aftrsn_memory`** — banque de marque AFTER SUN PEOPLE.
- **`memory_registry`** — table de routage : `code, table_name, display_name, kind('shared'|'brand'), active`.
- Schéma de banque : `id, title, content, collection, knowledge_type, source, source_ref, content_hash UNIQUE, embedding VECTOR(1536), created_at, updated_at` ; index HNSW cosine.
- Embeddings : OpenAI `text-embedding-3-small` (credential `OpenAi account`). DB : credential `Postgres account`.
- Tags `collection` marque : `canon`, `voice`, `ops`, `process`, `history`.

## 2. Ajouter une nouvelle marque

1. Choisir un **code** lowercase court et stable (ex. `aftrsn`).
2. Créer la table `CREATE TABLE <code>_memory (... schéma standard ...);` + index HNSW.
3. Insérer dans `memory_registry` : `(code, '<code>_memory', '<Nom marque>', 'brand', true)`.
4. (Aucun nouveau workflow à créer si les templates sont paramétrés — voir §4.)
5. Ingérer le contenu de marque (§3).

## 3. Ingérer du contenu dans une banque

1. Préparer la source (wiki GitHub `wiki/brand/...` pour la marque ; `wiki/architecture/...` pour lumina).
2. Lancer l'ingestion **paramétrée** avec le `code` cible (résolu en table via le registre).
3. Le flux : (Truncate si reload complet) → fetch → chunk → **Embed (OpenAI)** → **Build Insert paramétré** (`$1..$n`, `md5($2)` pour `content_hash`, `$k::vector`) → **Postgres Insert** (Options → Query Parameters = `{{ $json.values }}`) → **Verify Table Count**.
4. Vérifier le count par `source_ref`.

⚠️ L'ingestion fait un **TRUNCATE** au début si configurée en reload complet → ne relancer que si prêt à reconstruire la table. Un échec après le truncate vide la table.

## 4. Règles de routage & sécurité

- Les workflows/MCP reçoivent un **code**, puis **résolvent la table via `memory_registry`** (allowlist). **Jamais** d'interpolation d'un nom de table arbitraire (injection SQL).
- MCP : `search_brand_memory(brand=code, question)` → table de la marque ; `search_process_memory(question)` → `lumina_memory`.
- Recherche : `SELECT ... FROM <table> [WHERE collection = $coll] ORDER BY embedding <=> $vec::vector LIMIT 5`.

## 5. Câblage des agents

- Gardien-marque, Contenu-canaux, agents de domaine → banque **de la marque** (`aftrsn_memory`).
- Agents dev/ops → `lumina_memory`.
- L'outil mémoire des agents s'appelle **`Knowledge`** (cohérent node ↔ prompt).

## 6. Dette / à faire connue

- [ ] Migrer `asp_memory` → `lumina_memory`.
- [ ] Créer `aftrsn_memory` + `memory_registry`.
- [ ] Templatiser ingestion + recherche (paramètre = code→table via registre).
- [ ] Ingérer la **bible** dans `aftrsn_memory` (`collection='canon'`).
- [ ] Brancher le filtrage `collection` dans la recherche du Gardien-marque.
- [ ] MCP routé par marque (+ durcir l'auth, actuellement none).

## 7. Challenges & leçons (mémo)

- Mémoire mélangée + recherche sans filtre → agent noyé. → frontière physique par marque.
- Table par marque sans templates → N pipelines. → templates paramétrés + registre + MCP routé.
- Nom de table paramétré → allowlist registre (anti-injection).
- Inserts → paramétrés ; nom d'outil mémoire identique node↔prompt ; valider end-to-end (appel MCP).
