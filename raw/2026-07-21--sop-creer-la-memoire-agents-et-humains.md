---
type: raw
title: "SOP_creer-la-memoire-agents-et-humains"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/sop_creer-la-memoire-agents-et-humains.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

---
title: "SOP — Créer la mémoire (agents + humains) depuis le wiki Git"
type: sop
status: active
tags: [n8n, pgvector, notion, github, obsidian, memory, idempotence]
version: 1.1
last_review: 2026-06-25
---

# SOP — Créer la mémoire (agents + humains) depuis le wiki Git

## Objectif
À partir d'**un wiki en markdown dans Git** (la source de vérité), alimenter **automatiquement** deux mémoires dérivées : **la mémoire des AGENTS** (pgvector, pour la recherche sémantique) et **la vue des HUMAINS** (Notion, pour lire/partager). Les deux sont **parallèles, idempotentes, régénérables** — jamais en chaîne, jamais éditées à la main.

## Vue d'ensemble (hub parallèle)
```
   Obsidian (coffre + graphe)
        │  plugin Obsidian Git : commit + push auto
        ▼
   GitHub  wiki/  (SOURCE DE VÉRITÉ)
        │
   ┌────┴──────────────────────────┐
   ▼ n8n + OpenAI embeddings        ▼ n8n + Notion API
 pgvector (mémoire AGENTS)        Notion (vue HUMAINS)
 idempotent: TRUNCATE + reload    idempotent: upsert par fichier
```
> **Règle d'or :** Git = source unique. pgvector et Notion se **régénèrent** depuis Git. On ne les édite jamais directement.

---

## Partie 1 — Obsidian : le coffre & la synchro vers GitHub
**À quoi ça sert / pourquoi ici.** Obsidian est le point de départ humain de toute la connaissance : c'est là que le fondateur écrit, organise et **visualise** le savoir avant qu'il n'irrigue le reste du système. On commence par cette partie parce que rien ne peut être mémorisé ni publié tant que ça n'existe pas, proprement, dans le coffre. Le coffre Obsidian est **physiquement le même dossier que le repo Git**, ce qui garantit une source de vérité unique et versionnée. Le plugin Obsidian Git **pousse automatiquement** chaque changement vers GitHub, et c'est ce push qui réveille toute la chaîne en aval (Parties 2 à 4). La **vue graphe** et les **backlinks** servent à comprendre comment les idées se relient — un avantage qui deviendra crucial à mesure que la base grossit.
**Accès (gouvernance).** Pour l'instant, **seul le fondateur** a accès à Obsidian. Plus tard, à mesure que l'équipe grandit et que les membres comprennent mieux le fonctionnement du business, l'accès sera ouvert — d'abord aux **chefs de département**.

**Éléments et leur rôle :**
- **Vault Obsidian = repo Git** — mêmes fichiers (`raw/`, `wiki/`, `CLAUDE.md`, `index.md`, `log.md`) ; on ouvre le dossier du repo via *Open folder as vault*.
- **Plugin Obsidian Git** — commit + push automatiques (~10 min). C'est lui qui **déclenche** les pipelines mémoire (Partie 4).
- **`raw/` vs `wiki/`** — `raw/` = sources brutes curées ; `wiki/` = pages propres maintenues par le LLM (Claude Code). On ingère/publie le `wiki/`, pas `raw/`.
- **Accès distant (optionnel)** — Obsidian sur Coolify (image `linuxserver/obsidian`, port 3000). ⚠️ La synchro passe par **Git** (PAS CouchDB / LiveSync).

---

## Partie 2 — Mémoire des AGENTS (wiki → pgvector)
**À quoi ça sert / pourquoi ici.** Cette partie transforme le wiki (du texte lisible) en **mémoire interrogeable par les agents IA**. Elle vient après la Partie 1 parce qu'elle **consomme** ce qui a été validé dans le wiki : sans wiki propre, rien à ingérer. Le rôle de pgvector est de permettre aux agents de retrouver la bonne information **par le sens** (recherche sémantique), pas par mots-clés. On **reconstruit la table à chaque run** (TRUNCATE + reload) pour que la mémoire soit toujours un miroir exact du wiki, sans doublons ni résidus. C'est cette mémoire qui rend les agents réellement utiles : ils répondent à partir de la connaissance de l'entreprise, pas de généralités. C'est donc le **socle opérationnel** sur lequel reposeront tous les agents (superviseur + spécialistes).

**Workflow : `LUMINA — Memory — Ingestion (Wiki→pgvector)`** — lit tout le `wiki/` et le charge dans pgvector ; idempotent.

**Chaîne et rôle de chaque node :**
1. **Trigger** — démarre le workflow (manuel pour tester, webhook en production).
2. **Truncate** (`TRUNCATE asp_memory;`) — vide la table avant de recharger → garantit un miroir propre (l'idempotence).
3. **HTTP Request** (GitHub Trees API, `git/trees/main?recursive=1`) — récupère la **liste de tous les fichiers** du repo en un appel.
4. **Split Out** (`tree`) — éclate cette liste en **un item par fichier** pour les traiter un à un.
5. **Filter** (`type=blob` ET `path` commence par `wiki/` ET finit par `.md`) — ne garde que les pages du wiki (exclut config, `raw/`, dossiers).
6. **Get a file** (`File Path = {{ $json.path }}`, **As Binary OFF**) — récupère le **contenu réel** de chaque fichier.
7. **Set Document** — met le contenu en forme + métadonnées attendues (`document`, `title`, `source`, `source_ref`, `collection`, `knowledge_type`).
8. **Chunk** (Code, `for … of $input.all()`) — découpe chaque document en **morceaux digestes** pour l'embedding (boucle sur TOUS les docs).
9. **Embed (OpenAI `text-embedding-3-small`)** — transforme chaque morceau en **vecteur** (sa signature sémantique, 1536 dim).
10. **Build Insert** — prépare la requête SQL d'insertion.
11. **Postgres Insert** (`{{ $json.query }}`) — écrit les vecteurs dans `asp_memory`.

**Vérif :** `SELECT source_ref, count(*) FROM asp_memory GROUP BY source_ref;` → uniquement des `wiki/…`.

---

## Partie 3 — Vue des HUMAINS (wiki → Notion)
**À quoi ça sert / pourquoi ici.** Cette partie crée la version **lisible et partageable** de la même connaissance, pour les humains. Elle est **parallèle** (pas séquentielle) à la Partie 2 : les deux dérivent du même wiki mais ne se parlent jamais. On les sépare parce qu'agents et humains ont des **besoins différents** — les agents veulent de la recherche sémantique, les humains veulent une lecture confortable, du partage et du mobile. La publication est **idempotente par fichier** (on archive l'ancienne page, on recrée la fraîche), pour que Notion reste propre sans accumuler de doublons. Notion est volontairement **optionnel et non porteur** : le système tourne sans lui, c'est une commodité humaine. Elle prendra toute son importance quand **l'équipe grandira** et que plusieurs personnes devront consulter la connaissance sans toucher à la technique.

**Workflow : `PUBLISH-NOTION`** — publie/met à jour les pages Notion à partir du `wiki/` ; upsert par fichier.

**Chaîne et rôle de chaque node :**
1. **Build Notion payload** (Code) — construit la page Notion à partir du fichier wiki ; pose l'**URL Source** comme clé de correspondance (`payload.properties.Source.url`).
2. **Query Notion DB BY SOURCE** — cherche si une page existe **déjà** pour ce fichier (recherche par l'URL Source). ⚠️ Corps du filtre en **Expression + `JSON.stringify`** ; header **`Notion-Version: 2022-06-28`**.
3. **Split Out** (`results`) — éclate les résultats pour pouvoir traiter l'ancienne page trouvée.
4. **Archive page** (`PATCH /v1/pages/{id}` `{"archived": true}`) — **archive l'ancienne version** (c'est ce qui évite les doublons).
5. **Create page** — **recrée la page à jour** dans la base Notion.

> Configs **exhaustives** node par node de PUBLISH-NOTION : voir `lumina-systeme-reference.md` (v1.2).
> **IDs :** Notion DB `6fac8cb891e44c749e37a86419826905` · data_source `657827e3-1e2a-48d5-8f52-5954e3618687` · intégration **AFTRN-n8n**.

---

## Partie 4 — Automatisation (un seul déclencheur)
**À quoi ça sert / pourquoi ici.** Cette dernière partie **relie tout** pour que la synchronisation se fasse seule, sans intervention. Elle vient en dernier parce qu'elle **suppose que les Parties 2 et 3 existent déjà et sont idempotentes** — on n'automatise que ce qui marche déjà à la main. Le déclencheur est un **push GitHub** sur le `wiki/` : dès que la connaissance change, les deux pipelines se relancent **en parallèle**. Comme les deux cibles sont idempotentes, re-déclencher à chaque push est **totalement sûr** (jamais de doublon ni de corruption). Résultat : le fondateur écrit dans Obsidian, et quelques secondes plus tard, **agents et humains ont tous la version à jour**, sans toucher à n8n. C'est ce qui transforme un assemblage de workflows en un **système vivant et autonome**.

**Éléments et leur rôle :**
- **Webhook GitHub** (repo → Settings → Webhooks, event = push, secret) — notifie n8n à chaque changement.
- **Webhook Trigger n8n** (`LUMINA — Orchestrateur — Sync`) — reçoit la notification ; filtre branche `main` + chemins `wiki/**`.
- **2× Execute Workflow** — lance **en parallèle** l'Ingestion (Partie 2) et le Publish-Notion (Partie 3).

---

## Pré-requis
- Repo GitHub avec `wiki/*.md` (voir SOP « Intake GitHub idempotent » pour remplir le wiki).
- PostgreSQL + pgvector ; table `asp_memory` (`embedding vector(1536)`, index HNSW cosinus).
- Credentials n8n : **GitHub**, **OpenAI**, **Notion**. Embedding **figé** : `text-embedding-3-small` (1536).

## Erreurs fréquentes (vécues)
- **Un seul doc ingéré** → Chunk en `$input.first()`. Corriger en `for … of $input.all()`.
- **404 sur Get a file** → File Path en *Fixed* au lieu d'*Expression*.
- **Pas de texte / base64** → `As Binary` resté ON sur Get a file.
- **Notion : 0 résultat au filtre** → corps non passé en *Expression + JSON.stringify*.
- **Doublons Notion** → pas d'archivage de l'ancienne page, ou matching sur le mauvais champ.
- **Node Anthropic n8n = 404** → toujours passer par **HTTP Request** pour Claude.

## Points critiques
- **Idempotence** : pgvector = TRUNCATE+reload ; Notion = archive+recrée par fichier.
- **Embedding figé** à 1536 dim.
- Ne **jamais** éditer pgvector ou Notion à la main (dérivés).

## Adaptation à une autre marque
Même pattern : changer owner/repo, la base Notion, la collection. Les deux pipelines sont réutilisables tels quels ; chaque marque a son propre coffre.

## Version
v1.1 — 2026-06-25 (renumérotation Parties 1-4 + descriptions par partie/node + gouvernance d'accès Obsidian)
