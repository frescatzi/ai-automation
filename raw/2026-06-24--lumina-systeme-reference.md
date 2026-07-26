---
type: raw
title: "lumina-systeme-reference"
source_url: "drive:1Gi77FBEDXYaP7ASjJuKvNUlbDigHO-Td"
captured: 2026-06-24
vault: brands
brand: AFTRSN
immutable: true
---

# LUMINA — Système de connaissance (référence complète)

> AI Operating System d'AFTER SUN PEOPLE (AFTRSN). Modèle **hub parallèle** : `Git (source de vérité) → pgvector` **ET** `Git → Notion`, en parallèle, jamais en chaîne.
> État : **complet et fonctionnel de bout en bout** (2026-06-23).

---

## 1. Vue d'ensemble (4 couches)

```
   Drive Inbox ──(robot Gemini)──▶ GitHub raw/ ──(Claude Code)──▶ wiki/
                                                      │
                        ┌──────────────────────────────┴──────────────┐
                        ▼ n8n + OpenAI                                 ▼ n8n + Notion API
                   pgvector (mémoire AGENTS)                    Notion (vue HUMAINS)
                   MCP search_brand_memory / webhook            contenu formaté, par marque
```

- **Source de vérité = Git** (repos `frescatzi/ai-automation`, `brands`, `personal` + `lumina-meta`). pgvector et Notion = **dérivés régénérables, jamais édités à la main**.
- **Embedding figé** : OpenAI `text-embedding-3-small` (1536 dim).
- **Piège** : node Anthropic natif n8n interdit (bug 404) → HTTP Request.

---

## 2. Couche 1 — Intake (robot d'ingestion) ✅ — **v2 (2026-06-24)**

Workflow `LUMINA-INTAKE-ROUTER/DRIVE→GITHUB` : draine l'inbox Drive → OpenAI classe → écrit dans `<coffre>/raw/` sur GitHub → supprime de Drive. **Multi-fichiers** (vide tout ce qui traîne à chaque run).

```
Schedule Trigger → Search files and folders (liste TOUT l'inbox)
   → Download file → Extract from File → OPEN-AI-CALL (classification JSON)
   → Code_Parse → If Rejected → Create a file (GitHub) → Delete a file (Drive)
```

- **Schedule Trigger** : déclenche périodiquement (ex. Every Hour) → l'inbox se vide toute seule. Workflow **Active**.
- **Search files and folders** (Google Drive, search: fileFolder) : liste **tous** les fichiers du dossier **Lumina Inbox** (`1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`), Return All ON. → sort N items. C'est le « videur » : il traite le backlog ET les nouveaux.
- **Download file** : File ID `{{ $json.id }}` (chaque item de la liste).
- **OPEN-AI-CALL** (HTTP) : `POST https://api.openai.com/v1/chat/completions`, auth OpenAI, body `model: gpt-4o-mini`, `temperature: 0`, `response_format: {type: json_object}`, messages system (le classeur Lumina : brands / ai-automation / personal / reject) + user `{{ JSON.stringify($json.data) }}`. Sortie : `choices[0].message.content` = JSON `{vault, brand, reason}`.
- **Code_Parse** (Code, **Mode = Run Once for Each Item**) : lit `$json.choices[0].message.content`, récupère le fichier apparié via `$('Search files and folders').item.json` et le texte via `$('Extract from File').item.json.data`, construit `path` (`raw/AAAA-MM-JJ--slug.md`, ou `<brand>/raw/…` si vault=brands), le frontmatter `type: raw`, et sort `{ vault, brand, reject, repo, path, fileContent, fileId, filename }`.
- **If Rejected** : `reject = true` → ne range pas.
- **Create a file** (GitHub) : repo `By URL` = `https://github.com/frescatzi/{{ $json.repo }}`, path `{{ $json.path }}`, content `{{ $json.fileContent }}`. **On Error = Stop Workflow** (garde-fou).
- **Delete a file** (Drive) : File ID `{{ $('Code_Parse').item.json.fileId }}` (le bon fichier, par appariement).
- **Sécurité** : Delete **après** un Create réussi → un fichier n'est jamais supprimé de Drive sans avoir été écrit sur GitHub. Si l'écriture échoue, le fichier reste → re-traité au prochain run.

### Pièges résolus (intake v2, 2026-06-24)
1. **Gemini 503** (free tier surchargé, même avec retry) → bascule du classifieur sur **OpenAI** (gpt-4o-mini, coût négligeable). Adapter Code_Parse : `choices[0].message.content` (OpenAI) au lieu de `candidates[0].content.parts[0].text` (Gemini).
2. **1 seul fichier traité** → « Fetch Test Event » (test) ne donne qu'1 item. Fix : node **Search files and folders** qui liste tout l'inbox.
3. **Code_Parse ne sortait qu'1 item** → était en `Run Once for All Items` avec `.first()`/`.all()[0]`. Fix : **Run Once for Each Item** + `$json` + `$('…').item` (appariement).
4. **Delete « Referenced node doesn't exist »** → le node de liste s'appelle `Search files and folders`, pas `Google Drive`. Référence finale : `$('Code_Parse').item.json.fileId`.
5. **GitHub 422 « sha wasn't supplied »** = le fichier existe déjà (backlog déjà rangé). Pour vider : `On Error = Continue` le temps d'un run, **puis remettre `Stop Workflow`**. En régime normal (fichier rangé puis supprimé), pas de collision.

### Marche à suivre — recréer le robot d'intake (SOP courte)
1. **Schedule Trigger** (Every Hour).
2. **Google Drive → Search files and folders** : dossier Lumina Inbox, Return All ON, `trashed = false`.
3. **Google Drive → Download file** : File ID `{{ $json.id }}`, mode Expression.
4. **Extract from File** : Extract From Text File, Input Binary Field `data`.
5. **HTTP → OPEN-AI-CALL** : config ci-dessus (chat completions, json_object). Activer **Retry On Fail**.
6. **Code → Code_Parse** : Mode **Run Once for Each Item**, code ci-dessus.
7. **IF → If Rejected** : `{{ $json.reject }}` true/false.
8. **GitHub → Create a file** : repo By URL, path, content ; **On Error = Stop Workflow**.
9. **Google Drive → Delete a file** : File ID `{{ $('Code_Parse').item.json.fileId }}`.
10. Tester en **Execute workflow**, puis **Activer** le workflow.

## 3. Couche 2 — Rédaction wiki (Claude Code) ✅

`raw/ → wiki/` : Claude Code en local (`~/Documents/Lumina AI/GitHub/ai-automation`), guidé par `CLAUDE.md`. Produit `concept-*` / `synthese-*`, backlinks, `index.md`, `log.md`. Commit + push. Validation humaine : `status: draft → active`.

## 4. Couche 3 — Mémoire des agents (pgvector) ✅

Workflow `LUMINA — Memory — Ingestion (Wiki→pgvector)` : Trigger → **Truncate `asp_memory`** → HTTP GitHub Trees API → Split Out → Filter `wiki/*.md` → Get a file → Set Document → Chunk → Embed (OpenAI 1536) → Build Insert → Postgres Insert. **Idempotent** (TRUNCATE + reload). Récupération : MCP `search_brand_memory` + webhook.
- Table `asp_memory` : `id, content, source, source_ref, metadata JSONB, embedding VECTOR(1536), created_at`. Index HNSW cosinus.

## 5. Couche 4 — Vue humaine (Notion) ✅ — NOUVEAU

Base Notion **« AI Automation — Knowledge »** (teamspace AI-AUTOMATION).
- DB id : `6fac8cb891e44c749e37a86419826905` · data_source : `657827e3-1e2a-48d5-8f52-5954e3618687`
- Propriétés : `Page` (title), `Type` (select concept/synthese/sop), `Source` (url GitHub), `Tags` (multi), `Vault` (select), `Updated` (date).
- ⚠️ Intégration n8n **AFTRN-n8n** doit être **ajoutée en Connection** sur la base (sinon « database not found »).

Workflow `LUMINA-MEMORY-INGESTION/PUBLISH-NOTION` — **idempotent (upsert par fichier)** ✅ :
```
Trigger → Query Notion DB → HTTP GitHub Trees → Split Out1 (tree) → Filter (wiki/*.md, exclut README)
        → Get a file → Set Document → Code (Parsing)
              ├─→ Build Notion payload → POST Notion          (créer la version fraîche)
              └─→ Query BY SOURCE → Split Out (results) → ARCHIVE PAGE   (archiver l'ancienne)
```
- **Idempotence = upsert par fichier** : pour chaque page wiki, on cherche sa page Notion par l'URL Source, on archive l'ancienne (`PATCH /v1/pages/{id}` body `{"archived":true}`), et on recrée la fraîche. → jamais de doublon, sûr (on ne touche jamais « tout Notion »), compatible futur webhook GitHub par fichier.
- **Code (Parsing)** : extrait frontmatter (notionTitle/type/tags/vault/sourceUrl/updated/body), isole le corps. ⚠️ **doit avoir une SEULE entrée (Set Document)**.
- **Build Notion payload** : markdown → **blocs Notion** (heading_1/2/3, bulleted/numbered, quote, paragraph ; nettoyage `**`/`` ` ``/liens ; coupe 1990 car/bloc ; max 95 blocs) + propriétés. C'est lui qui produit l'**URL Source complète** (`payload.properties.Source.url`).
- **POST Notion** : `POST /v1/pages`, auth Notion API, header `Notion-Version: 2022-06-28`, body `{{ $json.payload }}`.
- **Query BY SOURCE** : `POST /v1/databases/{id}/query`, body en **mode Expression** : `{{ JSON.stringify({ filter: { property: "Source", url: { equals: $json.payload.properties.Source.url } } }) }}`. Branché sur **Build Notion payload** (qui a l'URL exacte écrite dans Notion).
- **ARCHIVE PAGE** : `PATCH /v1/pages/{{ $json.id }}` body `{"archived":true}`, On Error = Continue.

### Pièges résolus (2026-06-24)
1. **Fil parasite `Query Notion DB → Code (Parsing)`** : injectait 1 item non-wiki dans Code (Parsing) → page fantôme (titre vide, URL `…/blob/main/`). Fix : Code (Parsing) ne reçoit QUE Set Document.
2. **Expression `{{ }}` non évaluée** dans le body JSON du filtre → envoyée en texte brut → 0 résultat. Fix : body en **mode Expression** + `JSON.stringify`.
3. **`sourceUrl` de Code (Parsing) tronqué** (`…/blob/main/`) → ne matche pas Notion. Fix : filtrer sur `payload.properties.Source.url` (Build payload), pas sur `sourceUrl`.
4. **README.md** inclus par le Filter → exclu via condition `path does not contain README`.
5. **Race archive/create** : ne jamais lancer la branche archive seule ; Git reste la source → re-remplir Notion = 1 run publish.

---

## 6. ⏭️ À faire (prochaines étapes)

1. ✅ **Dédup / idempotence du publish Notion** — **FAIT (2026-06-24)**. Upsert par fichier (archive l'ancienne + recrée la fraîche). 2 runs = 12 pages stables, zéro doublon.
2. **Automatiser les déclencheurs** : webhook GitHub push → ingestion pgvector + publish Notion auto.
3. **Répliquer** `brands` et `personal` (vaults + bases Notion) quand ils auront du contenu.
4. **Filtre publication** : n'envoyer vers Notion que `status: active` (décommenter la ligne dans le Code parse).

---

## 7. Décisions figées (mémo)
- Source de vérité = Git ; dérivés régénérables, jamais édités main.
- Embedding = `text-embedding-3-small` (1536). Jamais node Anthropic natif.
- Idempotence : pgvector = TRUNCATE + reload ; Notion = **upsert par fichier** (archive ancienne + recrée).
- Notion : body du filtre toujours en **mode Expression** + `JSON.stringify` ; matcher sur `payload.properties.Source.url`.
- Un node Code à entrée unique : ne jamais y brancher un 2ᵉ flux (sinon items parasites).
- Nommage : `LUMINA — <Process> — <Détail>` (infra) ; `AFTRSN — …` (marque).
- Sécurité : workflow planifié+publié obsolète = **dépublier**. Tâches critiques → gate CEO.

*v1.2 — 2026-06-24 (intake v2 : classifieur OpenAI + drain multi-fichiers + SOP de recréation).*
