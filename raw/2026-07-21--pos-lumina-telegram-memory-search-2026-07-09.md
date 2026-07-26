---
type: raw
title: "POS-LUMINA_Telegram-Memory-Search_2026-07-09"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_telegram-memory-search_2026-07-09.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS — LUMINA-TELEGRAM · M3 MEMORY-SEARCH (EXACT)

**Version :** v1.0
**Date :** 2026-07-09
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra »
**Étape couverte :** M3 — MEMORY-SEARCH. Brancher les intents `memory_search` + `general_question` sur la recherche vectorielle, avec **rédaction LLM** de la réponse.
**Boussoles :** SPEC-BUILD M3 v0.2 · SPEC-BUILD Telegram v0.2 · Bible 360° v1.9. **n8n fait foi.**

---

## 1. Objectif

Donner à Lyra l'accès à la mémoire LUMINA : quand le Router classe un message en `memory_search` (ou `general_question`), la Gateway appelle un sous-workflow qui interroge la brique **memory-search** (recherche vectorielle Postgres), puis un **LLM rédige** une réponse courte, claire et **sourcée** (grounding strict) renvoyée dans Telegram.

**Critères de succès (Karter), validés live le 09-07 :** réponses **cohérentes · fiables · rapides**.

## 2. Prérequis (acquis)

1. **Gateway WF1** `LUMINA-TELEGRAM-GATEWAY` — `uGF3pSv6aTK79cqi` (Personal, Active).
2. **Router WF2** `LUMINA-TELEGRAM-INTENT-ROUTER` — `KTLwQi7ZKHDTexMZ` : renvoie `{ intent, brand, …, query, chat_id, message_id }`.
3. **Brique memory-search** `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` — `ZhUeKCo8nMp35gk1` : `POST https://n8n.aftersunpeople.com/webhook/memory-search`.
4. **Sentinel** `xzaH0uWy0idKVphF` (Error Workflow).
5. Credential **`LUMINA-Lyra - Telegram`** (bot @luminaLyraBot) ; credential **`OpenAi account`** (LLM de rédaction).

## 3. Contrat memory-search (figé après inspection directe du node `Build Search`, 09-07)

Body accepté :
- **`question`** (string) — obligatoire (embedding en amont).
- **`brand`** (alias `bank`) — table `<brand>_memory`, **défaut `lumina`**.
- **`limit`** — `Math.min(parseInt(limit)||5, 20)` → **défaut 5, plafond 20**.
- **`collection`** — si fourni → `WHERE collection = '<coll>'` ; **si omis → `WHERE collection IS DISTINCT FROM 'episodic'`** (toute la banque sauf le journal épisodique brut).

Réponse : tableau de `{ id, title, extrait (200 car.), collection, similarite (0–1, arrondi 4) }`, trié par proximité vectorielle.

**Choix M3 :** on envoie **`{ question, brand, limit: 5 }`**, sans `collection` (défaut = episodic exclu → fiabilité).

## 4. Procédure — Sous-workflow `LUMINA-TELEGRAM-MEMORY-SEARCH`

**Workflow réel :** ID **`AzEakDoVK7T9EXG2`** · projet **Personal** · tags `BRAIN · SHARED · TEST · ASSISTANT-BOT` · **Settings → Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH`** (`xzaH0uWy0idKVphF`). *(Publié ; sans conséquence car déclenché par Execute Workflow, pas de webhook externe.)*

Flux : `Receive the request → Clean the question → Query the memory bank → Prepare context → Any result?` → **TRUE** `Write the answer (LLM) → Shape reply` / **FALSE** `Write "not found"`.

### N1 — `Receive the request`
- **Execute Workflow Trigger** (`executeWorkflowTrigger`), **Input Source = Accept all data**. Reçoit `{ query, brand, chat_id, message_id }`.

### N2 — `Clean the question`
- **Code**, Mode = **Run Once for All Items**, JavaScript :
```javascript
/**
 * N2 — Clean the question (memory-search)
 * Rôle   : nettoie la question et sécurise la marque avant l'appel HTTP.
 * Entrée : { query, brand, chat_id, message_id } (depuis la Gateway).
 * Sortie : { question, brand, chat_id, message_id }.
 * Note   : brand vide/none/karter/agency -> 'lumina' (défaut sûr).
 */
const q = ($json.query ?? '').trim();
let brand = ($json.brand ?? 'lumina');
if (['none','karter','agency',''].includes(String(brand).toLowerCase())) brand = 'lumina';
return [{ json: { question: q, brand, chat_id: $json.chat_id, message_id: $json.message_id } }];
```

### N3 — `Query the memory bank`
- **HTTP Request** (`httpRequest` v4.2), **POST** `https://n8n.aftersunpeople.com/webhook/memory-search`.
- **Authentication = None** · **Send Body = ON** · **Body Content Type = JSON** · **Specify Body = Using JSON**.
- **JSON** (mode expression, commence par `{{`) :
```
{{ JSON.stringify({ question: $json.question, brand: $json.brand, limit: 5 }) }}
```
- **Options → Response Format = JSON** · **Timeout = 15000 ms**.
- ⚠️ Body via `JSON.stringify` (piège figé v1.8).

### N4 — `Prepare context`
- **Code**, Mode = **Run Once for All Items** : rassemble les résultats (robuste à toutes les formes de sortie du HTTP node), construit le contexte numéroté (top 5) et compte les résultats.
```javascript
let items = $input.all().map(i => i.json);
if (items.length === 1) {
  const j = items[0];
  if (Array.isArray(j))           items = j;
  else if (Array.isArray(j.body)) items = j.body;
  else if (Array.isArray(j.data)) items = j.data;
}
const ctx = $('Clean the question').first().json;
const valid = items.filter(r => r && r.title);
const context = valid.slice(0, 5).map((r, i) =>
  `[${i+1}] ${r.title} (${r.collection ?? 'canon'}) : ${String(r.extrait ?? '').replace(/\s+/g,' ')}`
).join('\n');
return [{ json: { chat_id: ctx.chat_id, question: ctx.question, count: valid.length, context } }];
```

### N5 — `Any result?`
- **IF**, condition **Number** : `{{ $json.count }}` **is greater than** `0`.

### N6 (TRUE) — `Write the answer (LLM)`
- **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`), **Prompt = Define below** :
```
Tu es Lyra, l'assistante de Karter. Réponds à la question UNIQUEMENT à partir des extraits de mémoire ci-dessous.
Règles : langage simple et clair, 2 à 4 phrases, ton direct et utile. Si les extraits ne répondent pas, dis franchement que tu n'as rien trouvé de fiable — n'invente jamais. Cite les titres sources entre parenthèses.

Question : {{ $json.question }}

Extraits de mémoire :
{{ $json.context }}
```
- **Sous-node modèle** : **OpenAI Chat Model** (`lmChatOpenAi`) · credential **`OpenAi account`** · **Model = `gpt-5-mini`** (From list) · **Use Responses API = ON** · température non fixée (défaut). *(Décision 09-07 : garder gpt-5-mini — rapide/économique, réponse validée.)*
- Sortie du LLM : texte sous **`text`**.

### N7 (TRUE) — `Shape reply`
- **Edit Fields (Set)** v3.4, **Include Other Input Fields = OFF**, valeurs en **mode Expression** :
  - `chat_id` = `{{ $('Clean the question').first().json.chat_id }}`
  - `text` = `{{ $json.text }}`

### N8 (FALSE) — `Write "not found"`
- **Edit Fields (Set)**, **Include Other Input Fields = OFF** :
  - `chat_id` (**Expression**) = `{{ $json.chat_id }}`
  - `text` (**Fixed**) = `Je n'ai rien trouvé de fiable en mémoire sur cette question.`

Sortie du sous-workflow (branche exécutée) = **`{ chat_id, text }`**.

## 5. Procédure — Branchement Gateway (dispatch)

Sur la sortie **FALSE de `IF clarification`** :

1. **`Is it a memory question?`** (IF) — conditions **String**, combinateur **OR** : `{{ $json.intent }}` = `memory_search` **OU** = `general_question`.
2. Branche **TRUE** :
   - **`Prepare Search Input for Memory-Search`** (Set, Keep Only, valeurs en **Expression**) : `query`, `brand`, `chat_id`, `message_id`.
   - **`Run the memory search`** (Execute Sub-workflow → `LUMINA-TELEGRAM-MEMORY-SEARCH` `AzEakDoVK7T9EXG2`, **Wait for completion = ON**).
   - **`Reply to user`** (Telegram Send, credential `LUMINA-Lyra - Telegram`) : Chat ID `{{ $json.chat_id }}`, Text `{{ $json.text }}`, **Append n8n Attribution = OFF**, **Parse Mode = None**.
3. Branche **FALSE** → **`Reply (module pending)`** (stub existant, inchangé).

## 6. Vérification / recette (live Telegram, validé 09-07)

- `/memory lumina c'est quoi Hermes Exec ?` → réponse rédigée, claire, **sourcée** (`synthese-lumina-ai-os.md`, `concept-memoire-vivante-agents.md`) ; le LLM signale honnêtement l'absence de définition explicite (pas d'invention). ✅
- `Quel temps fait-il à Zurich ?` → routé `web_search` → stub « Module en construction ». ✅ (dispatch mémoire vs web correctement séparé)
- Aucune erreur JSON, Sentinel non déclenché, aucun « can't parse entities » (Parse Mode None). ✅

## 7. Difficultés rencontrées

1. **MCP n8n cloisonné** sur le projet AFTRSN → le wf memory-search `ZhUeKCo8nMp35gk1` et les workflows Telegram (projet Personal) **invisibles** par MCP → impossible d'inspecter `Build Search` par cette voie.
2. **Réponses brutes illisibles** : la recherche vectorielle seule renvoyait une concaténation d'extraits (titres + morceaux) — aucun sens pour un humain.
3. **`Bad Request: chat not found`** (Telegram 400) au node `Reply to user` : `chat_id` arrivait **vide** (en fait : la chaîne littérale `{{ $('Prepare context')… }}` non évaluée).

## 8. Solutions implémentées

1. **Inspection via navigateur** (Chrome) du node `Build Search` → contrat figé (`question, brand, limit` défaut 5/max 20, `collection` défaut exclut `episodic`). MCP laissé de côté pour ce workflow.
2. **Ajout d'une étape de rédaction LLM** (le « G » du RAG) : `Prepare context` construit un contexte sourcé, puis `Write the answer (LLM)` (gpt-5-mini) rédige une réponse **bridée aux extraits** (grounding) ; `Any result?` court-circuite vers un **fallback strict** si aucun résultat.
3. **Deux corrections** au node `Shape reply` :
   - valeurs passées en **mode Expression** (en Fixed, un `{{ }}` n'est **pas** évalué dans Set v3.4 → texte brut) ;
   - `chat_id` tiré via **`$('Clean the question').first()`** au lieu de `$('Prepare context').item` (la référence *paired item* `.item` n'est plus fiable **après un node LangChain**).

## 9. Lessons learned

1. **Set v3.4 : toute valeur contenant `{{ }}` doit être en mode Expression**, sinon elle part en texte littéral. *(Piège figé — Bible.)*
2. **Après un node LangChain, `$('X').item` n'est plus fiable** → utiliser `.first()` (ou lire un node situé avant le LLM). *(Piège figé — Bible.)*
3. **RAG = retrieval + generation** : une recherche vectorielle seule ne suffit pas côté UX ; il faut un node LLM de rédaction **bridé aux extraits** pour une réponse humaine, claire et fiable.
4. **Le grounding fait office de fact-check** : inutile d'empiler Maestro/Hermes/passe de vérif pour une Q&R en lecture (latence ×3–5, gain quasi nul). 1 seul appel LLM = cohérent, fiable, rapide.
5. **memory-search** : le défaut `collection` (episodic exclu) protège la fiabilité — le journal brut ne contamine pas les réponses.
6. Rappels figés réappliqués : body HTTP via `JSON.stringify` ; sortie Telegram Attribution OFF + Parse Mode None.

## 10. Références

- SPEC-BUILD M3 (v0.2) ; SPEC-BUILD Telegram v0.2 ; Bible 360° v1.9 (addendum M3).
- Workflows : sous-wf `LUMINA-TELEGRAM-MEMORY-SEARCH` **`AzEakDoVK7T9EXG2`** ; Gateway `uGF3pSv6aTK79cqi` ; Router `KTLwQi7ZKHDTexMZ` ; memory-search `ZhUeKCo8nMp35gk1` ; Sentinel `xzaH0uWy0idKVphF`.
- Générique associé : `POS-GENERIQUE_Branche-memory-search-telegram.md`.
- Convention de nommage des nodes (verbe d'action EN) : addendum Bible v1.9.

---

*POS EXACT M3 v1.0 — 2026-07-09. Validé live (cohérent · fiable · rapide). n8n fait foi.*
