---
type: raw
title: "AFTRSN-LUMINA — Etape 2 — DDL pret a executer (memory_registry + aftrsn_memory) + plan rename supervise"
source_url: "drive:16wyxgaqrvPqNMM_mfDRnheBH4FvugHtQ"
captured: 2026-07-02
vault: brands
brand: AFTRSN
immutable: true
---

-- =============================================================================
-- ÉTAPE 2 — Mémoire multi-banques : DDL PRÊT À EXÉCUTER
-- Cible : Postgres + pgvector (Hetzner/Coolify)
-- Exécuter via : un node Postgres "Execute Query" (n8n) OU psql.
-- Credential DB : "Postgres account".
-- =============================================================================
-- PRINCIPE ANTI-CASSE (important) :
--   On NE renomme PAS asp_memory tout de suite. Le registre découple le CODE
--   logique ('lumina') de la TABLE physique ('asp_memory'). Donc on enregistre
--   'lumina' -> 'asp_memory' : RIEN ne casse, la mémoire actuelle continue de
--   tourner. Le renommage physique devient un nettoyage cosmétique OPTIONNEL
--   (voir ÉTAPE 2b, supervisée).
-- Cette partie (1 à 3) est ADDITIVE, IDEMPOTENTE et NON destructive.
-- =============================================================================

-- 0) Prérequis
CREATE EXTENSION IF NOT EXISTS vector;

-- 1) Registre de routage (table de config, source unique de vérité)
CREATE TABLE IF NOT EXISTS memory_registry (
  code         TEXT PRIMARY KEY,
  table_name   TEXT NOT NULL,
  display_name TEXT,
  kind         TEXT NOT NULL CHECK (kind IN ('shared','brand')),
  active       BOOLEAN DEFAULT true,
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- Seed. 'lumina' pointe sur la table physique ACTUELLE (asp_memory) -> non-cassant.
INSERT INTO memory_registry (code, table_name, display_name, kind) VALUES
  ('lumina', 'asp_memory',    'Automation & Process (shared)', 'shared'),
  ('aftrsn', 'aftrsn_memory', 'AFTER SUN PEOPLE',              'brand')
ON CONFLICT (code) DO NOTHING;

-- 2) Banque de marque AFTER SUN PEOPLE (table vide, même schéma que la banque standard)
CREATE TABLE IF NOT EXISTS aftrsn_memory (
  id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title          TEXT,
  content        TEXT NOT NULL,
  collection     TEXT,            -- canon | voice | ops | process | history
  knowledge_type TEXT,
  source         TEXT,            -- 'GitHub' | 'Notion' | ...
  source_ref     TEXT,            -- ex. 'brands/aftrsn/wiki/genres.md'
  content_hash   TEXT UNIQUE,     -- md5(content) -> upsert idempotent
  embedding      VECTOR(1536),    -- OpenAI text-embedding-3-small
  created_at     TIMESTAMPTZ DEFAULT now(),
  updated_at     TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS aftrsn_memory_embedding_hnsw
  ON aftrsn_memory USING hnsw (embedding vector_cosine_ops);
CREATE INDEX IF NOT EXISTS aftrsn_memory_collection_idx     ON aftrsn_memory (collection);
CREATE INDEX IF NOT EXISTS aftrsn_memory_knowledge_type_idx ON aftrsn_memory (knowledge_type);

-- 3) Vérifications (lecture seule)
SELECT code, table_name, kind, active FROM memory_registry ORDER BY kind, code;
SELECT count(*) AS aftrsn_rows FROM aftrsn_memory;


-- =============================================================================
-- ÉTAPE 2b — RENOMMAGE PHYSIQUE asp_memory -> lumina_memory  (OPTIONNEL, SUPERVISÉ)
-- =============================================================================
-- ⚠️ CASSANT le temps de repointer les workflows. NE PAS exécuter sans, dans la
--    foulée, mettre à jour TOUTES les références à asp_memory dans n8n.
-- Grâce au registre, ce renommage n'est PAS nécessaire pour avancer — purement
--    cosmétique (faire correspondre nom physique et convention). À faire à tête
--    reposée, ensemble.
--
-- ALTER TABLE asp_memory RENAME TO lumina_memory;
-- UPDATE memory_registry SET table_name = 'lumina_memory' WHERE code = 'lumina';
--
-- Puis dans n8n (republier après) :
--   - LUMINA-RETRIEVAL-MEMORY/WEBHOOK -> node "Build Search" : FROM asp_memory  ->  FROM lumina_memory
--   - LUMINA-MEMORY-INGESTION/WIKI-GITHUB->PGVECTOR : Truncate / Build Insert / Verify Table Count -> lumina_memory
--   - Tout autre workflow référençant asp_memory.
--
-- ROLLBACK :
--   ALTER TABLE lumina_memory RENAME TO asp_memory;
--   UPDATE memory_registry SET table_name = 'asp_memory' WHERE code = 'lumina';
--   (et revert les workflows)
-- =============================================================================
