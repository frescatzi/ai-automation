---
type: raw
title: "Session_2026-06-25_Pipeline-raw-wiki_Commandes-Claude-Code"
source_url: "drive:1Ic1UIIs0Ono9cmUxp7Nr5nq3hqdN3b4N"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# Session — 25 juin 2026
## Pipeline `raw → wiki` & commandes Claude Code (`/ingest`, `/sync`)

### Objet de la session
Clarifier où le LLM intervient dans le système Lumina, comprendre pourquoi « le LLM ne s'exécute
pas automatiquement », et se doter de commandes Claude Code réutilisables pour compiler le wiki et
synchroniser le vault.

---

### 1. Le point clé : Obsidian n'exécute aucun LLM
Obsidian est juste un éditeur de fichiers `.md` posé au-dessus du dépôt Git. Quand on dit qu'Obsidian
est le « cerveau », on parle de l'endroit où la **connaissance est stockée**, pas d'un endroit qui
*réfléchit*. Stockage et calcul sont séparés.

Qui « pense » (LLM) dans la chaîne, et où :

| # | Étape | LLM | Où | Auto / manuel |
|---|-------|-----|----|---------------|
| 1 | Dépôt d'un doc dans Drive (Lumina Inbox) | — | Google Drive | manuel (normal) |
| 2 | Intake : tri + Drive→GitHub | gpt-4o-mini | n8n | ✅ auto |
| 3 | `raw/ → wiki/` : compiler les notes | **Claude (Claude Code)** | Mac local | ⚠️ **manuel** |
| 4 | Push `wiki/` sur GitHub | — | GitHub | semi-auto |
| 5 | Ingestion `wiki/ → pgvector` | embeddings OpenAI | n8n | ✅ auto (cross-trigger) |
| 6 | Publication `wiki/ → Notion` | — | n8n | ✅ auto (cross-trigger) |
| 7 | Réponses des 4 agents | Claude (agents) | n8n, via MCP `search_brand_memory` | ✅ auto |

→ Tout est automatique **sauf l'étape 3**. C'est le seul maillon manuel, et il n'est pas « dans
Obsidian » : c'est Claude Code, en local, en amont.

---

### 2. Comment se fait le transfert `raw → wiki`
Ce n'est **pas une copie ni un script** : c'est une **réécriture faite par le LLM**. `raw/` reste
intact (immutable) ; Claude Code le lit, le comprend et rédige des pages propres dans `wiki/`.

Analogie : `raw/` = notes brutes / matière première ; `wiki/` = articles d'encyclopédie rédigés à
partir de ces notes ; Claude Code = le rédacteur.

Ce que fait `/ingest` :
1. Lit `CLAUDE.md` (conventions).
2. Lit `index.md` + `log.md` (ce qui est déjà traité).
3. Scanne `raw/`, repère le nouveau / modifié.
4. Extrait les faits durables, les range par **sujet** dans la bonne page wiki (mise à jour ou
   création), avec les `[[liens]]`.
5. Pose les backlinks, fusionne les doublons.
6. Met à jour `index.md`, ajoute une ligne datée à `log.md`.

À retenir : ce n'est **pas du 1-pour-1** (un fichier `raw/` peut nourrir plusieurs pages wiki, et
inversement), et ça ne se déclenche **jamais tout seul**.

---

### 3. Récupérer ce qui est sur GitHub (pull)
Pas besoin de « connecter Claude à GitHub » : le vault Obsidian **est déjà** un clone local du dépôt
GitHub (`~/Documents/Lumina AI/GitHub/ai-automation`). « Récupérer » = faire un `git pull`. Dès le
pull, les fichiers apparaissent dans Obsidian (même dossier).

Boucle complète : `git pull` → `/ingest` → commit + push (Obsidian Git) → cross-trigger →
pgvector + Notion se rafraîchissent automatiquement. Tout reconverge.

⚠️ Rappel : **pgvector ne reflète que `wiki/`, pas `raw/`**. Des fichiers restés dans `raw/` ne sont
pas « complets » dans la mémoire des agents tant qu'ils ne sont pas compilés en pages wiki puis
poussés.

---

### 4. Livrables de la session — deux commandes Claude Code
À déposer dans `~/Documents/Lumina AI/GitHub/ai-automation/.claude/commands/` :

- **`/ingest`** (`ingest.md`) — compile ce qui attend dans `raw/` → `wiki/`, avec une étape backlinks
  dédiée (liens réciproques, pas de lien rouge, audit des pages orphelines). À utiliser quand le
  vault est déjà à jour.
- **`/sync`** (`sync.md`) — fait d'abord `git pull` puis enchaîne la compilation `/ingest`. À utiliser
  quand on veut d'abord rapatrier ce qui est sur GitHub.

Les deux sont écrites en anglais (convention « contenu opérationnel en anglais »), s'appuient sur
`CLAUDE.md`, et ne committent / pushent pas (versioning géré via Obsidian Git).

Lancer Claude Code **depuis la racine du vault** pour qu'il voie `raw/`, `wiki/`, `CLAUDE.md`.

---

### 5. Point ouvert / à vérifier
Confirmer l'état réel : ce qui est dans `raw/` vs `wiki/` vs pgvector, pour s'assurer qu'il n'y a pas
de décalage (notamment si des fichiers semblent déjà en mémoire agents alors qu'ils ne seraient que
dans `raw/`).
