---
type: raw
title: "POS-GENERIQUE_Branche-memory-search-telegram"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_branche-memory-search-telegram.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — Brancher une recherche mémoire (RAG) sur un bot de messagerie (GÉNÉRIQUE)

**Version :** v1.0
**Date :** 2026-07-09
**Objet :** patron réutilisable pour donner à un bot l'accès à une **banque mémoire vectorielle**, avec **rédaction LLM** d'une réponse humaine, claire et sourcée. Transposable à toute marque / tout bot / toute brique de recherche.

---

## 1. Objectif

Quand un routeur d'intention classe un message en `memory_search` (ou question générale de connaissance), la passerelle (`<GATEWAY>`) appelle un sous-workflow qui : interroge la brique de recherche vectorielle `<SEARCH_ENDPOINT>`, puis fait **rédiger** la réponse par un LLM **bridé aux extraits** (grounding), et la renvoie à l'utilisateur.

**Principe directeur :** peu importe la question, la réponse doit être **propre, claire, précise et fiable** pour l'humain. => RAG = *retrieval* (recherche) **+** *generation* (rédaction LLM).

## 2. Prérequis

1. `<GATEWAY>` : workflow qui authentifie et dispatche, avec un **stub** pour les intents non implémentés.
2. `<ROUTER>` : renvoie `{ intent, brand, query, chat_id, message_id, … }`.
3. `<SEARCH_ENDPOINT>` : `POST <URL>` acceptant `{ question, brand[, limit, collection] }` et renvoyant un tableau `{ id, title, extrait, collection, similarite }`.
4. `<ERROR_WF>` (filet d'erreurs global) ; `<BOT_CREDENTIAL>` ; `<LLM_CREDENTIAL>`.

## 3. Vérifier le contrat de la brique de recherche (AVANT de coder)

Inspecter le node qui construit la requête (souvent un Code node « Build Search ») pour figer :
- champs acceptés (`question`, `brand`/`bank`, `limit`, `collection`) et leurs **valeurs par défaut** ;
- filtre par défaut (ex. exclure la collection « journal brut » type `episodic` pour la fiabilité) ;
- forme exacte de la réponse.
> Si l'accès API/MCP au workflow est cloisonné, l'inspecter directement dans l'éditeur (navigateur). **Le live fait foi.**

## 4. Sous-workflow `<BOT>-MEMORY-SEARCH`

Nouveau workflow, appelé par la passerelle en `executeWorkflow`. Error Workflow = `<ERROR_WF>`.

Flux : `Receive the request → Clean the question → Query the memory bank → Prepare context → Any result?` → **TRUE** `Write the answer (LLM) → Shape reply` / **FALSE** `Write "not found"`.

1. **`Receive the request`** — Execute Workflow Trigger, *Accept all data* (`{ query, brand, chat_id, message_id }`).
2. **`Clean the question`** — Code (Run Once for All Items) : trim la question, marque sûre (valeurs ambiguës → `<DEFAULT_BRAND>`), repropage `chat_id`/`message_id`.
3. **`Query the memory bank`** — HTTP POST `<SEARCH_ENDPOINT>`, **Body JSON via `JSON.stringify({ question, brand, limit: <N> })`**, Response Format JSON. *(Body dynamique = toujours `JSON.stringify`.)*
4. **`Prepare context`** — Code : rassembler les résultats **quelle que soit la forme de sortie du HTTP node** (`$input.all()`, puis déballer un éventuel tableau / `.body` / `.data`), filtrer les entrées valides, construire un **contexte numéroté et sourcé** (top k) + un **compteur**.
5. **`Any result?`** — IF `count > 0`.
6. **`Write the answer (LLM)`** (branche TRUE) — LLM Chain, **Prompt défini**, système :
   > *« Réponds UNIQUEMENT à partir des extraits fournis. Langage simple et clair, 2–4 phrases. Si l'info n'y est pas, dis-le franchement, n'invente rien. Cite les titres sources. »*
   Modèle `<LLM_MODEL>` (un modèle rapide suffit pour reformuler), température basse.
7. **`Shape reply`** (TRUE) — Set : `chat_id` et `text` **en mode Expression** ; tirer `chat_id` d'un node **situé avant le LLM** (`$('Clean the question').first()...`).
8. **`Write "not found"`** (branche FALSE) — Set : message de repli honnête (pas d'invention).

Sortie du sous-workflow = `{ chat_id, text }`.

## 5. Branchement dans la passerelle

Sur la sortie du dispatch (là où se trouve le stub) :
1. **`Is it a memory question?`** (IF) : `intent ∈ { memory_search, general_question }` (OR).
2. **TRUE** → `Prepare <search> input` (Set : `query, brand, chat_id, message_id`, **Expression**) → `Run the memory search` (Execute Sub-workflow, **Wait ON**) → `Reply to user` (envoi bot ; **attribution OFF**, **parse mode neutre**).
3. **FALSE** → stub existant (autres intents).

## 6. Recette

1. Question connue → réponse rédigée, claire, **sourcée**.
2. Question hors banque → repli « rien trouvé de fiable » (pas d'invention).
3. Question d'un autre domaine (web, etc.) → reste au stub (dispatch correct).
4. Latence perçue rapide (1 appel recherche + 1 appel LLM).

## 7. Difficultés rencontrées

1. **Accès cloisonné** à la brique de recherche (API/MCP limité à un projet) → inspection impossible par cette voie.
2. **Réponse brute illisible** : la recherche seule renvoie des extraits concaténés.
3. **Message non délivré** (`chat not found` / destinataire vide) : identifiant de conversation perdu dans la chaîne.

## 8. Solutions implémentées

1. **Inspecter la brique dans l'éditeur** (navigateur) pour figer le contrat ; le live fait foi.
2. **Ajouter l'étape de rédaction LLM** bridée aux extraits + fallback via un IF `Any result?`.
3. **Corriger le node Set** : valeurs `{{ }}` en **mode Expression** ; tirer l'identifiant de conversation d'un node **avant le LLM** avec `.first()` (les *paired items* `.item` ne survivent pas à un node LangChain).

## 9. Lessons learned

1. **Node Set : valeur `{{ }}` = mode Expression obligatoire** (sinon texte brut).
2. **Après un node LangChain, `$('X').item` n'est plus fiable** → `.first()` ou node avant le LLM.
3. **RAG = recherche + rédaction** : ne jamais renvoyer les extraits bruts à l'utilisateur ; un LLM de rédaction **grounded** garantit une réponse humaine et fiable.
4. **Le grounding = fact-check intégré** : pour une Q&R en lecture, une seule passe LLM suffit ; empiler des agents/vérifications ralentit sans gain.
5. **Défaut de collection** = exclure le « journal brut » protège la fiabilité des réponses.
6. Rappels transverses : body HTTP dynamique via `JSON.stringify` ; sortie bot sans attribution auto et sans parse mode agressif.

## 10. Paramètres à substituer

`<GATEWAY>` · `<ROUTER>` · `<SEARCH_ENDPOINT>` · `<BOT>` / `<BOT_CREDENTIAL>` · `<LLM_CREDENTIAL>` / `<LLM_MODEL>` · `<DEFAULT_BRAND>` · `<N>` (limit) · `<ERROR_WF>` · `<CHAT_ID>`.

---

*POS GÉNÉRIQUE v1.0 — 2026-07-09. Dérivé de l'implémentation LUMINA-TELEGRAM M3 (memory-search + rédaction LLM).*
