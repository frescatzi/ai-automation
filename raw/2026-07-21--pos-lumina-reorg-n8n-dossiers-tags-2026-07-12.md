---
type: raw
title: "POS-LUMINA_Reorg-n8n-dossiers-tags_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_reorg-n8n-dossiers-tags_2026-07-12.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — Réorganisation n8n : dossiers, tags, marques (à valider avant exécution)

**Date :** 2026-07-12
**Contexte :** Projet n8n unique « Personal » (multi-projets = Enterprise, indisponible) → la séparation **socle / marques / personnel se fait par dossiers**. 65 workflows au total (la Bible v2.4 en fichait 32 → à resynchroniser en v2.5).
**Décisions actées :** Design 1 (Maestro = superviseur PAR marque ; le socle n'a PAS d'agent superviseur ; Lyra = passerelle d'entrée Telegram). Descriptions de dossiers stockées **dans la Bible ET en README** dans chaque dossier n8n.

---

## 1. Cible : arborescence + description de chaque dossier (1 dossier = 1 process)

### 01-LUMINA — 🌐 SOCLE partagé (plomberie ; AUCUN agent superviseur)
Le socle qui donne vie à tout le reste : entrée des données, mémoire, briques IA partagées, automatisation, connecteurs, interface. Partagé par toutes les marques.

| Sous-dossier | Process (README) |
|---|---|
| `LUMINA-01-INTAKE` 📥 | Entrée des données brutes : Drive→GitHub, archivage raw→Drive. |
| `LUMINA-02-MEMORY` 🧠 | Connaissance partagée : ingestion wiki→pgvector, recherche vectorielle, écriture mémoire, consolidation nocturne, provisioning de marque, keep-warm, health-check. |
| `LUMINA-03-BRAIN` 🤖 | Briques IA **partagées** : Router de modèles (LUMINA-AI-Router) + exécuteur (LUMINA-Hermes-Agent). **Pas de superviseur ici.** |
| `LUMINA-04-AUTOMATION` 🔄 | Orchestration automatique : sync Git→mémoire/Notion, auto-ingest raw→wiki. |
| `LUMINA-05-CONNECT` 🔌 | Exposition vers l'extérieur : serveur MCP. |
| `LUMINA-06-INTERFACE` 📱 | **(nouveau)** Interface d'entrée = bot Telegram **Lyra** (Gateway, Intent-Router, Memory-Search, Web-Hermes). Passerelle qui authentifie et route vers le bon superviseur (marque ou perso). |
| `LUMINA-08-UTIL` 🛠️ | Utilitaires manuels d'admin (SQL/DDL). Jamais actifs. |

### 02-AFTRSN — 🌞 Marque AFTER SUN PEOPLE
| Sous-dossier | Process |
|---|---|
| `AFTRSN-02-MEMORY` 🧠 | Ingestion de la mémoire propre à AFTRSN. |
| `AFTRSN-03-BRAIN` 🤖 | Superviseur **AFTRSN-Maestro** + ses 7 sous-agents + outils marque. |

### 03-KARTER-AMARIS · 04-KOWRIS · 05-LUMINA-AGENCY — 🏷️ Marques (à peupler)
Vides pour l'instant. Chaque marque recevra, via `LUMINA-BRAND-PROVISION` + `cloneRoster` : sa banque mémoire `<code>_memory`, son `<MARQUE>-Maestro` et ses sous-agents. (`05-LUMINA-AGENCY` = agence AI, **nom exact à confirmer**.) KOWRIS = marketplace.

### 06-PERSONNEL — 🧑 Environnement personnel
Tout ce qui n'est pas une marque : connaissances, notes personnelles, lectures, idées, formations, + (à venir) ton agent personnel.

### Hors chaîne
`Z_ARCHIVES` ⚫ (dépréciés `ZZZ_`, ne jamais réactiver) · `SandBox` 🔵 (essais jetables).

---

## 2. Rangement des workflows « en vrac » (racine → dossier cible)

| Workflow | Dossier cible |
|---|---|
| LUMINA-TELEGRAM-GATEWAY | 01-LUMINA / LUMINA-06-INTERFACE |
| LUMINA-TELEGRAM-INTENT-ROUTER | 01-LUMINA / LUMINA-06-INTERFACE |
| LUMINA-TELEGRAM-MEMORY-SEARCH | 01-LUMINA / LUMINA-06-INTERFACE |
| LUMINA-TELEGRAM-WEB-HERMES | 01-LUMINA / LUMINA-06-INTERFACE |
| LUMINA-MEMORY-KEEPWARM | 01-LUMINA / LUMINA-02-MEMORY |
| LUMINA-HEALTHCHECK-MEMOIRE | 01-LUMINA / LUMINA-02-MEMORY |
| LUMINA - Auto-Ingest - Raw→Wiki | 01-LUMINA / LUMINA-04-AUTOMATION |
| AFTRSN-Knowledge-Capture | 02-AFTRSN / AFTRSN-02-MEMORY |
| AFTRSN-Marketing-Planner | 02-AFTRSN / AFTRSN-03-BRAIN |

### À archiver dans `Z_ARCHIVES` (temporaires de debug — je ne peux pas les supprimer, seulement archiver ; suppression = toi)
`ZZZ_20260708_DBuser` · `ZZZ_20260705_tmp_sql_runner` · `ZZZ_20260705_tmp_mem_probe` · `ZZZ_20260705_tmp_w4_probe` · `ZZZ_20260705_tmp_budget_test` · `ZZZ_20260705_tmp_skill_write`

> Les ~32 workflows déjà rangés dans les sous-dossiers `LUMINA-0X-*` / `AFTRSN-0X-*` (cf. Bible v2.4) sont **déjà à leur place** — je vérifierai chaque dossier en direct pendant l'exécution et signalerai tout intrus.

---

## 3. Tags (4 axes) à poser / compléter

**Axes :** Domaine (📥 INTAKE · 🧠 MEMORY · 🤖 AI-AGENTS · 🧭 ROUTER · ⚙️ EXEC · 🔄 automation · 🔌 MCP · 🛠️ UTIL · 👩🏽‍💻 ASSISTANT-BOT) · Statut (🟢 PROD · 🔵 TEST · ⚫ DEPRECATED) · Déclencheur (🪝 webhook · ⏰ scheduled · ⚡ EVENT · 👆 MANUAL) · Marque (🌞 AFTRSN · 🌐 SHARED ; + à créer : 🏷️ KARTER-AMARIS, KOWRIS, LUMINA-AGENCY, 🧑 PERSO).

| Workflow (sans tags aujourd'hui) | Tags proposés |
|---|---|
| LUMINA-TELEGRAM-GATEWAY | 🟢 PROD · 👩🏽‍💻 ASSISTANT-BOT · 🌐 SHARED · 🪝 webhook |
| LUMINA-TELEGRAM-INTENT-ROUTER | 🟢 PROD · 👩🏽‍💻 ASSISTANT-BOT · 🧭 ROUTER · 🌐 SHARED |
| LUMINA-TELEGRAM-MEMORY-SEARCH | 🟢 PROD · 👩🏽‍💻 ASSISTANT-BOT · 🧠 MEMORY · 🌐 SHARED |
| LUMINA-TELEGRAM-WEB-HERMES | 🟢 PROD · 👩🏽‍💻 ASSISTANT-BOT · ⚙️ EXEC · 🌐 SHARED |
| LUMINA-MEMORY-KEEPWARM | 🟢 PROD · ⏰ scheduled · 🧠 MEMORY · 🌐 SHARED |
| LUMINA-HEALTHCHECK-MEMOIRE | 🟢 PROD · ⏰ scheduled · 🧠 MEMORY · 🌐 SHARED |
| LUMINA - Auto-Ingest - Raw→Wiki | 🟢 PROD · 🪝 webhook · 🔄 automation · 🌐 SHARED |
| AFTRSN-Knowledge-Capture | 🟢 PROD · 🧠 MEMORY · 🌞 AFTRSN |
| AFTRSN-Marketing-Planner | 🟢 PROD · 🤖 AI-AGENTS · 🌞 AFTRSN |
| ZZZ_* (les 6) | ⚫ DEPRECATED |

---

## 4. Corrections documentaires (pour la Bible v2.5)
- **`LUMINA-Hermes-Exec` → `LUMINA-Hermes-Agent`** : déjà le vrai nom dans n8n ; corriger la Bible.
- **Cadrage rôles :** *Lyra* = passerelle d'entrée Telegram (socle) ; *Maestro* = superviseur PAR marque ; le socle n'a pas de superviseur.
- **Roster de marques :** AFTRSN · Karter Amaris · KOWRIS · Lumina Agency (nom à confirmer) · + PERSONNEL (non-marque).
- **Décompte :** 32 → **65 workflows** ; recompter et re-ficher.

---

## 5. Ordre d'exécution (après ta validation)
1. Déplacer les 9 workflows en vrac (§2) vers leurs dossiers.
2. Archiver les 6 `ZZZ_` dans `Z_ARCHIVES`.
3. Poser/compléter les tags (§3) + revue rapide des tags sur les 65.
4. Créer un README (workflow désactivé + Sticky Note) dans chaque dossier (§1).
5. Resync Bible v2.5 + double sauvegarde (projet Claude + Drive).
6. Vérification finale : aucun `Call` de Maestro cassé, publiés toujours publiés.
