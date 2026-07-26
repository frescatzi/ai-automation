---
type: raw
title: "GUIDE-BUILD_LUMINA-M4-Web-Hermes-Search_2026-07-09"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m4-web-hermes-search_2026-07-09.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# GUIDE-BUILD — LUMINA-TELEGRAM · M4 WEB / HERMES SEARCH

**Version :** v1.0 (guide de build intermédiaire — Karter exécute dans n8n, valide ; POS produits après validation)
**Date :** 2026-07-09
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra »
**Étape :** M4 — brancher les intents `web_search` + `hermes_ops` sur l'exécuteur **Hermès**, avec **rédaction gpt-5-mini** (grounding strict).
**Boussoles :** SPEC-BUILD M4 v0.2 · POS M3 v1.0 · Bible 360° v1.9. **n8n fait foi.**

---

## 1. Vue d'ensemble

Deux chantiers :
- **Partie A** — créer le sous-workflow `LUMINA-TELEGRAM-WEB-HERMES` (appelle Hermès, fait rédiger la réponse, renvoie `{chat_id, text}`).
- **Partie B** — insérer la branche M4 dans le dispatch de la Gateway (`uGF3pSv6aTK79cqi`), juste après la branche mémoire, en remplacement du stub pour ces deux intents.

Contrats vérifiés live : **Hermès** (`Dwv4rcMqNAyQzlrF`) entrée `{ message, brand? }`, sortie `{ text, outcome, usedSkills }`.

---

## 2. Partie A — Sous-workflow `LUMINA-TELEGRAM-WEB-HERMES`

Nouveau workflow, projet **Personal**. **ID réel : `RRAZbO4zEb1sFNvb`.** Tags : `PROD · BRAIN · SHARED · ASSISTANT-BOT`. **Settings → Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH`** (`xzaH0uWy0idKVphF`).
> Note MCP (piège figé au build) : ce sous-wf a un **Execute Workflow Trigger** → **jamais visible en MCP**. Le MCP n'expose que les workflows **publiés + trigger ∈ {Schedule, Webhook, Form, Chat}**. Vérification par canvas/JSON, pas par MCP.

Flux : `Receive from Gateway → Prepare task → Call Hermes → Hermes OK?` → **TRUE** `Write the answer (LLM) → Shape reply` / **FALSE** `Write fallback`.

### N1 — `Receive from Gateway`
- **Execute Workflow Trigger** (`executeWorkflowTrigger`), **Input Source = Accept all data**. Reçoit `{ query, brand, chat_id, message_id }`.

### N2 — `Prepare task`
- **Code**, Mode = **Run Once for All Items** :
```javascript
/**
 * N2 — Prepare task (web/hermes)
 * Entrée : { query, brand, chat_id, message_id } (depuis la Gateway).
 * Sortie : { message, brand, chat_id, message_id }.
 * Note   : mappe query -> message ; marque sûre (défaut aftrsn = exécuteur AFTRSN).
 */
const message = ($json.query ?? '').trim();
let brand = ($json.brand ?? 'aftrsn');
if (['none','karter','agency',''].includes(String(brand).toLowerCase())) brand = 'aftrsn';
return [{ json: { message, brand, chat_id: $json.chat_id, message_id: $json.message_id } }];
```

### N3 — `Call Hermes`
- **Execute Sub-workflow** → workflow **`LUMINA-Hermes-Exec`** (`Dwv4rcMqNAyQzlrF`), **Wait for completion = ON**.
- **Workflow Inputs** (defineBelow) : `message` = `{{ $json.message }}`.
  > Note build : le trigger d'Hermès expose seulement `message` (jsonExample). Le `brand` n'est donc pas mappable sans toucher Hermès ; Hermès applique son défaut interne **`aftrsn`**, ce qui convient au MVP. Pour passer `brand` plus tard : basculer le trigger d'Hermès en « Accept all data » (petit changement, différé).
- Sortie attendue : `{ text, outcome, usedSkills }`.
- **Vérifier** qu'aucun timeout n8n n'est plus court que ~45 s (Hermès poll jusqu'à ~42 s).

### N4 — `Hermes OK?`
- **IF**, condition **Boolean** :
```
{{ $('Call Hermes').first().json.outcome === 'success' && String($('Call Hermes').first().json.text ?? '').trim().length > 0 }}
```
- **TRUE → N5** · **FALSE → N5'**

### N5 (TRUE) — `Write the answer (LLM)`
- **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`), **Prompt = Define below** :
```
Tu es Lyra, l'assistante personnelle de Karter (LUMINA OS / AFTER SUN PEOPLE).
Reformule pour Telegram la RÉPONSE D'HERMÈS ci-dessous : claire, concise, ton chaleureux et éditorial.
RÈGLES STRICTES : n'ajoute AUCUN fait, n'invente rien ; conserve tels quels chiffres, noms, sources et liens.
Si la réponse d'Hermès est déjà courte et nette, renvoie-la quasi telle quelle. Réponds en français.

RÉPONSE D'HERMÈS :
{{ $('Call Hermes').first().json.text }}
```
- **Sous-node modèle** : **OpenAI Chat Model** (`lmChatOpenAi`), credential **`OpenAi account`**, **Model = `gpt-5-mini`** (From list), **Use Responses API = ON**. Sortie du LLM sous **`text`**.
> Piège figé : lire Hermès via **`$('Call Hermes').first()`** — après un node LangChain, `.item` (paired item) n'est plus fiable.

### N5' (FALSE) — `Write fallback`
- **Edit Fields (Set)** v3.4, **Include Other Input Fields = OFF** :
  - `chat_id` (**Expression**) = `{{ $('Prepare task').first().json.chat_id }}`
  - `text` (**Fixed**) = `Je n'ai pas pu obtenir de réponse d'Hermès pour l'instant. Réessaie dans un moment.`

### N6 (après N5) — `Shape reply`
- **Code**, Mode = **Run Once for All Items** :
```javascript
const ctx = $('Prepare task').first().json;
let text = String($json.text ?? '').trim();
if (!text) text = "Je n'ai pas pu formuler de réponse. Réessaie dans un moment.";
text = text.replace(/\n{3,}/g, '\n\n');
if (text.length > 3800) text = text.slice(0, 3800) + '…';   // marge sous la limite Telegram (4096)
return [{ json: { chat_id: ctx.chat_id, text } }];
```

**Sortie du sous-workflow** (branche exécutée) = **`{ chat_id, text }`**.

---

## 3. Partie B — Câblage dans le dispatch de la Gateway

Rappel structure actuelle (post-M3) : `IF clarification` → **FALSE** → `Question for Memory-Search ?` → **TRUE** = branche mémoire / **FALSE** = `Reply (module pending)` (stub).

On insère M4 **sur la sortie FALSE de `Question for Memory-Search ?`** (avant le stub).

### B1 — `Is it a web/ops question?`
- **IF**, conditions **String**, combinateur **OR** :
  - `{{ $json.intent }}` **is equal to** `web_search`
  - `{{ $json.intent }}` **is equal to** `hermes_ops`
- Câbler l'**entrée** de ce IF sur la sortie **FALSE** de `Question for Memory-Search ?`.

> ⚠️ **Piège `$json` clobbering** : le node Telegram `Send typing` **remplace `$json`** par la réponse de l'API Telegram. Tous les nodes situés **après** `Send typing` doivent donc lire le contrat via **`$('<node qui appelle le Router>').item.json.<champ>`** (nom exact à confirmer dans la Gateway — probablement `Classify the intent`), **pas** `$json`. Ordre retenu : `Send typing` d'abord, puis `Prepare Input` qui repropage depuis le Router.

### B2 (TRUE) — `Send typing`
- **Telegram**, Resource = **Message**, Operation = **Send Chat Action**, credential **`LUMINA-Lyra - Telegram`**.
  - **Chat ID** = `{{ $('Classify the intent').first().json.chat_id }}`
  - **Action** = `typing`
> Effet : « Lyra écrit… ». Dure ~5 s (limite Telegram) ; sur un appel long l'indicateur s'efface avant la réponse — acceptable MVP (sinon voir §5 repli accusé texte).

### B3 — `Prepare Input for Web-Hermes`
- **Edit Fields (Set)**, **Keep Only Set Fields**, valeurs en **mode Expression**, repropagées depuis le Router (car `$json` = réponse de `Send typing`) :
  - `query` = `{{ $('Classify the intent').first().json.query }}`
  - `brand` = `{{ $('Classify the intent').first().json.brand }}`
  - `chat_id` = `{{ $('Classify the intent').first().json.chat_id }}`
  - `message_id` = `{{ $('Classify the intent').first().json.message_id }}`
> Le node qui suit (`Run the web/hermes search`, Execute Workflow **passthrough**) transmet **cet item** au sous-wf → il doit être le contrat propre, pas la réponse Telegram. D'où la repropagation ici.

### B4 — `Run the web/hermes search`
- **Execute Sub-workflow** → **`LUMINA-TELEGRAM-WEB-HERMES`** (`RRAZbO4zEb1sFNvb`), **Wait for completion = ON**. Sortie = `{ chat_id, text }`.

### B5 — `Reply to user (hermes)`
- **Telegram Send Message**, credential **`LUMINA-Lyra - Telegram`** :
  - **Chat ID** = `{{ $json.chat_id }}`
  - **Text** = `{{ $json.text }}`
  - **Append n8n Attribution = OFF** · **Parse Mode = None**

### B6 (FALSE) — stub inchangé
- La sortie **FALSE** de `Is it a web/ops question?` reste branchée sur **`Reply (module pending)`** (les autres intents attendent M5+).

---

## 4. Recette (test live depuis Telegram, bot Lyra)

1. `web_search` : « cherche les dernières tendances Afro House 2026 » → « Lyra écrit… » puis réponse rédigée, cohérente, sourcée. ✅
2. `hermes_ops` : une demande d'exécution/veille → compte-rendu clair d'Hermès, reformulé Lyra. ✅
3. **Fidélité** : la réponse ne contient aucun fait absent de la sortie d'Hermès (grounding OK). ✅
4. **Échec Hermès** (backend indispo) : message de repli propre, Sentinel non déclenché intempestivement. ✅
5. `memory_search` et clarification : inchangés. ✅
6. Aucune erreur `can't parse entities` (Parse Mode None), aucune erreur JSON.

---

## 5. Pièges à respecter (figés)

1. **Set v3.4** : toute valeur `{{ }}` en **mode Expression** (Fixed → texte littéral).
2. **Après un node LangChain**, lire via **`.first()`** (pas `.item`).
3. **Sortie Telegram** : **Attribution OFF** + **Parse Mode None**.
4. **Execute Sub-workflow** : **Wait for completion = ON** (sinon pas de `{chat_id,text}` en retour).
5. **Latence Hermès ~45 s** : vérifier les timeouts ; ne pas conclure à un échec trop tôt.
6. **Ne pas empiler** au-delà d'une passe de rédaction (pas de Maestro/vérif en plus) — piège latence M3.
7. **Repli si le typing ne suffit pas** : insérer un `Reply to user (ack)` (« Je m'en occupe, un instant… ») avant `Run the web/hermes search`, puis la réponse finale (2 messages).

---

## 6. Après validation

Produire et **double-sauvegarder** (projet Claude + Drive) :
- `POS-LUMINA_Telegram-Web-Hermes-Search_<date>.md` (exact) → Drive **POS-EXACT**.
- `POS-GENERIQUE_Branche-web-hermes-telegram.md` → Drive **POS-GENERIQUE**.
Puis bump **SPEC-BUILD M4 v0.3** (§ Écarts & décisions) + addendum **Bible v1.10** si structurant. IDs réels du sous-wf à consigner.

---

*GUIDE-BUILD M4 v1.0 — 2026-07-09. Karter construit dans n8n et valide. n8n fait foi.*
