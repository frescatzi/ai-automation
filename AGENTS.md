# AGENTS.md — Coffre `ai-automation`

> **Schéma de CE coffre.** Discipline le LLM-bibliothécaire pour la zone *IA & automation*.
> Le routage entre coffres vit dans `lumina-meta/ROUTING.md`. Ici = le *comment* : structure + workflows **Ingest / Q&A / Lint** propres à ce coffre.
> Coffre **autonome** (repo Obsidian séparé). Source de vérité = ces fichiers. GitHub = backup.

---

## 1. Périmètre du coffre

On garde ici tout savoir **réutilisable hors marque** sur l'IA et l'automation :
API, MCP, agents, prompting, n8n, embeddings, modèles (ChatGPT/Codex/Gemini), RAG, fine-tuning, webhooks, intégrations.

**Hors périmètre →** si c'est propre à une marque → coffre `brands`. Si c'est perso/business générique → coffre `personal`. En cas de doute, appliquer `lumina-meta/ROUTING.md`. Ne **jamais** conserver ici un topic hors périmètre « pour ranger plus tard ».

---

## 2. Structure interne

```text
ai-automation/
├── AGENTS.md     ← ce fichier (schéma + workflows)
├── index.md      ← carte de navigation du wiki (maintenue par le LLM)
├── log.md        ← journal append-only du coffre (ingests, Q&A, lints)
├── raw/          ← sources CURÉES et IMMUABLES (le LLM lit, ne modifie jamais)
└── wiki/         ← pages PROPRES (résumés, concepts, synthèses) — écrites par le LLM
```

- **`raw/`** : tu y déposes des sources que *tu* as choisies. Pas d'inbox en vrac. Immuable : on ajoute, on ne réécrit pas.
- **`wiki/`** : le LLM écrit ici. Tu lis. C'est la couche propre, validable, publiable.

---

## 3. Conventions (héritées de `lumina-meta/ROUTING.md`, précisées ici)

### 3.1 Nommage
- `raw/AAAA-MM-JJ--source-titre-court.md`
- `wiki/concept-nom.md` (page-concept) ou `wiki/synthese-sujet.md` (synthèse)

### 3.2 Frontmatter `raw/`
```yaml
---
type: raw
title: "Titre de la source"
source_url: "https://…  | export-Codex | export-chatgpt | note-manuelle"
captured: 2026-06-20
vault: ai-automation
brand: null
tags: [api, mcp]
immutable: true
---
```

### 3.3 Frontmatter `wiki/`
```yaml
---
type: wiki
title: "Nom du concept / synthèse"
status: draft          # draft | active | retired
publish: none          # none | notion
vault: ai-automation
brand: null
sources: [raw/2026-06-20--mcp-intro.md]
related: ["wiki/concept-agent.md"]   # réf. internes au coffre
updated: 2026-06-20
---
```

> Backlinks Obsidian `[[concept-nom]]` dans le corps. **Rappel : les liens ne sortent pas de ce coffre** — un renvoi vers une marque ou vers `personal` se fait en texte (`brands › AFTRSN › …`).

### 3.4 Statut & publication
- `status: draft|active|retired` (validation dans Obsidian).
- `publish: none|notion` (déclenche Notion → puis Postgres).
- **Atteint les agents seulement si `status: active` ET `publish: notion`.**

---

## 4. Workflow INGEST (déposer une source → enrichir le wiki)

**Déclencheur :** une nouvelle source dans `raw/`.

```
1. LIRE la source dans raw/ (ne jamais la modifier).
2. IDENTIFIER les concepts-clés qu'elle introduit ou enrichit.
3. POUR CHAQUE concept :
     - s'il existe déjà une page wiki/ → la mettre à jour (ajouter, nuancer, dater).
     - sinon → créer wiki/concept-nom.md (frontmatter status: draft).
4. RÉSUMER la source dans la/les page(s) concernée(s) : idée centrale, points utiles,
   limites/conditions d'usage. Citer la source dans `sources:`.
5. RELIER : ajouter les backlinks [[…]] internes + `related:` dans le frontmatter.
6. METTRE À JOUR index.md (ajouter le concept à la carte s'il est nouveau).
7. JOURNALISER dans log.md : une ligne `## [date] ingest | Titre`.
```

**Sortie attendue :** la source reste intacte dans `raw/` ; une ou plusieurs pages `wiki/` (en `draft`) la reflètent ; `index.md` et `log.md` à jour.

---

## 5. Workflow Q&A (répondre à une question depuis le coffre)

```
1. PARTIR de index.md pour localiser les pages pertinentes.
2. LIRE les pages wiki/ concernées (et au besoin les sources raw/ citées).
3. RÉPONDRE en t'appuyant sur le wiki ; signaler explicitement si l'info manque
   ou est en draft non validé.
4. REVERSER la valeur : si la réponse produit une synthèse réutilisable,
   l'écrire dans wiki/ (nouvelle page ou enrichissement) pour que le coffre grandisse.
5. JOURNALISER si la Q&A a modifié le wiki.
```

> Principe Karpathy : une bonne réponse **laisse une trace** dans le wiki, elle ne s'évapore pas dans le chat.

---

## 6. Workflow LINT (santé du coffre, périodique)

À lancer régulièrement (manuel ou planifié). Détecter et corriger :

```
- CONTRADICTIONS entre pages (même sujet, affirmations divergentes).
- PÉRIMÉ : pages dont le contenu est daté/obsolète → proposer status: retired.
- ORPHELINES : pages wiki/ sans aucun backlink entrant → relier ou archiver.
- CONCEPTS SANS PAGE : notions citées partout mais sans page dédiée → créer.
- RÉFS MANQUANTES : pages wiki/ sans `sources:` → retrouver la provenance.
- TROUS : sujets attendus mais absents → signaler (et proposer une recherche).
```

**Sortie :** un rapport bref + les corrections appliquées + une ligne `lint` dans `log.md`.

---

## 7. Gouvernance

- **Valider = promouvoir** une page de `draft` → `active` (revue par le CEO).
- **Publier** = passer `publish: notion` (déclenche la chaîne vers Postgres).
- **`raw/` immuable** : provenance préservée.
- **Gate CEO** : coûts, sécurité, clés API, suppression de mémoire.

---

## 8. `log.md` — format

Append-only, greppable :
```text
## [2026-06-20] ingest | Intro au Model Context Protocol
## [2026-06-20] lint   | 2 orphelines reliées, 1 page retired
## [2026-06-20] qa     | "Différence MCP vs function calling" → maj wiki/concept-mcp
```

---

*AGENTS.md `ai-automation` v2.2 — 2026-06-20. Modifier ce fichier = journaliser dans log.md.*
