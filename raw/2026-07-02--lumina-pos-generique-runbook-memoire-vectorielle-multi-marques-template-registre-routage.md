---
type: raw
title: "LUMINA — POS GENERIQUE — Runbook memoire vectorielle multi-marques (template + registre + routage)"
source_url: "drive:1cgfjLV_cp7yAd6u_C9UB4cKct4PlomKX"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Runbook : mémoire vectorielle multi-marques (template + registre + routage)

**Procédure Opérationnelle Standardisée réutilisable** — tout AI-OS multi-entités avec mémoire vectorielle + agents.

> Checklist concise et indépendante de l'implémentation. Numérotation à partir de 1.

---

## 1. Modèle de référence

- **Une banque partagée** (savoir réutilisable inter-entités).
- **Une banque par entité** (canon + ops propres). Frontière dure = la banque/entité ; frontières douces = tags internes (`collection`, `knowledge_type`).
- **Un registre** `(_code, table_name, display_name, kind, active_)` = source unique de vérité du routage.
- **Templates** d'ingestion et de recherche **paramétrés** par le code (résolu en table via le registre).

## 2. Ajouter une entité

1. Code lowercase court et stable.
2. Créer la banque (schéma standard + index HNSW).
3. Enregistrer dans le registre (`kind='brand'`).
4. Ingérer via le template paramétré. (Pas de nouveau workflow si templates en place.)

## 3. Ingérer

Source → (truncate si reload) → fetch → chunk → embed → **insert paramétré** (`$1..$n`, hash de contenu, `ON CONFLICT DO NOTHING`) → vérifier le count. Attention au pattern **TRUNCATE-puis-reload** (échec = banque vide).

## 4. Rechercher / router

- Entrée = `code` + question. Résoudre la table via le **registre** (allowlist). Embeder la question, k-NN `ORDER BY embedding <=> q LIMIT k`, filtre `collection` optionnel.
- Exposer **un** outil routé `search_entity_memory(entity, question)` + un pour la banque partagée.

## 5. Sécurité & gouvernance

- **Allowlist via registre** pour tout nom de table paramétré (anti-injection).
- Inserts **paramétrés** (jamais de concaténation).
- Auth du point d'accès (MCP/webhook) : pas de « no auth » hors MVP.
- Sauvegarde/idempotence : hash de contenu + `ON CONFLICT`.

## 6. Câblage des agents

Chaque agent lit la **bonne banque** (entité vs partagée). Nom de l'outil mémoire **identique** entre le node et le prompt. Sous-ciblage par tag `collection` si besoin.

## 7. Santé & maintenance

- Count par `source_ref` après chaque ingestion.
- Health-check périodique (embeddings + count) avec alerte.
- Purge/backup **par entité** (avantage clé des banques séparées).

## 8. Challenges & leçons

- Mémoire mélangée + recherche sans filtre → agent noyé. → frontière physique par entité.
- Granularité → selon l'**horizon opérationnel** (années, multi-entités, multi-plateformes), pas la simplicité MVP.
- Table par entité → risque N pipelines → **templates paramétrés + registre + outil routé** ; « ajouter une entité » = clone paramétré.
- Nom de table paramétré → **allowlist**. Inserts → paramétrés. Valider **end-to-end**.
