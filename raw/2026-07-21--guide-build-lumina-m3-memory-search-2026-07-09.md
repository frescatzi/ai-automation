---
type: raw
title: "GUIDE-BUILD_LUMINA-M3-Memory-Search_2026-07-09"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m3-memory-search_2026-07-09.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# GUIDE-BUILD — LUMINA TELEGRAM · M3 MEMORY-SEARCH

**Version :** v0.1 (guide de build, à exécuter dans n8n)
**Date :** 2026-07-09
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra »
**Étape :** M3 — brancher les intents `memory_search` + `general_question` sur la recherche vectorielle.
**Boussoles :** SPEC-BUILD M3 v0.1 · SPEC-BUILD Telegram v0.2 · Bible 360° v1.8. **n8n fait foi.**
**Mode de travail :** Claude fournit la marche à suivre, **Karter exécute dans n8n et valide**.

---

## 0. Ce que l'inspection a tranché (point ouvert §8.1 du SPEC-BUILD M3)

Node **Build Search** du wf memory-search `ZhUeKCo8nMp35gk1` (nom réel *LUMINA-RETRIEVAL-MEMORY*, projet Personal / dossier LUMINA-02-MEMORY) lu en direct. Contrat **définitif** :

- **`question`** (string) — obligatoire (embedding en amont).
- **`brand`** (alias `bank`) — table `<brand>_memory`, **défaut `lumina`**.
- **`limit`** — **accepté** : `Math.min(parseInt(limit)||5, 20)` → **défaut 5, plafond 20**.
- **`collection`** — **accepté** : si fourni → `WHERE collection = '<coll>'` ; **si omis → `WHERE collection IS DISTINCT FROM 'episodic'`** (on lit toute la banque sauf le journal épisodique brut).
- **Sortie** : tableau de `{ id, title, extrait (200 car.), collection, similarite (0–1, arrondi 4) }`, trié par proximité vectorielle.

**Décision de build :** on envoie **`{ question, brand, limit: 5 }`**, **sans** `collection` (le défaut exclut déjà `episodic` = fiabilité). Rapide (limit 5) + fiable (episodic écarté).

---

## 1. Sous-workflow `LUMINA-TELEGRAM-MEMORY-SEARCH`

Nouveau workflow, projet **Personal**, appelé par la Gateway en `executeWorkflow` (**non publié / non actif**).

### Étape 1 — Créer le workflow
1. Nouveau workflow → projet **Personal** → nom **`LUMINA-TELEGRAM-MEMORY-SEARCH`**.
2. Tags (facultatif, cohérence) : `ASSISTANT-BOT · BRAIN · SHARED`.
3. **Settings → Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH`** (`xzaH0uWy0idKVphF`).

### Étape 2 — N1 `Réception Gateway`
- Node **Execute Workflow Trigger** (« When Executed by Another Workflow »).
- **Input data mode = Accept all data.**
- Reçoit au minimum `{ query, brand, chat_id, message_id }`.

### Étape 3 — N2 `Prépare requête`
- Node **Code**, **Mode = Run Once for All Items**, Language JavaScript :
```javascript
const q = ($json.query ?? '').trim();
let brand = ($json.brand ?? 'lumina');
if (['none','karter','agency',''].includes(String(brand).toLowerCase())) brand = 'lumina'; // défaut sûr
return [{ json: {
  question: q,
  brand,
  chat_id: $json.chat_id,
  message_id: $json.message_id
} }];
```

### Étape 4 — N3 `Appel memory-search`
- Node **HTTP Request**.
- **Method = POST**, **URL = `https://n8n.aftersunpeople.com/webhook/memory-search`**.
- **Send Body = ON**, **Body Content Type = JSON**, **Specify Body = Using JSON**.
- Champ JSON en **mode expression** (icône `fx`), le contenu commence par `{{` (**pas** `={{`) :
```
{{ JSON.stringify({ question: $json.question, brand: $json.brand, limit: 5 }) }}
```
- **Response → Response Format = JSON.**
- ⚠️ **Piège figé v1.8** : body via `JSON.stringify`, jamais de JSON écrit à la main autour d'une valeur dynamique (casse sur `"`/`\`/saut de ligne).
- On **n'ajoute pas** `collection` (défaut = episodic exclu, voulu).

### Étape 5 — N4 `Formate réponse`
- Node **Code**, **Mode = Run Once for All Items**, Language JavaScript.
- Version **robuste** (améliore le SPEC v0.1 : gère toutes les formes de sortie possibles du node HTTP — tableau éclaté en items, ou 1 item contenant `[]` / `{data|body:[]}`) :
```javascript
// 1) rassembler toutes les entrées quelle que soit la forme de sortie du HTTP node
let items = $input.all().map(i => i.json);
if (items.length === 1) {
  const j = items[0];
  if (Array.isArray(j))            items = j;
  else if (Array.isArray(j.body))  items = j.body;
  else if (Array.isArray(j.data))  items = j.data;
  // sinon : 1 seul résultat -> on le garde tel quel
}

const ctx = $('Prépare requête').first().json;

// 2) rien de fiable -> pas d'invention
if (!items.length || !items[0] || !items[0].title) {
  return [{ json: { chat_id: ctx.chat_id,
    text: "Je n'ai rien trouvé de fiable en mémoire sur cette question." } }];
}

// 3) top 3, sourcé (titre + extrait), tronqué -> court et rapide
const top = items.slice(0, 3).map((r, i) =>
  `${i+1}. ${r.title} — ${String(r.extrait ?? '').replace(/\s+/g,' ').slice(0,180)}…`
).join('\n');
const text = `🔎 Mémoire (${items[0].collection ?? 'canon'}) :\n${top}`;

return [{ json: { chat_id: ctx.chat_id, text } }];
```

### Étape 6 — Câblage du sous-workflow
`Réception Gateway → Prépare requête → Appel memory-search → Formate réponse`.
Sortie finale = **`{ chat_id, text }`** (consommée par la Gateway).
**Save** (ne pas activer — c'est un sous-workflow appelé).

---

## 2. Branchement dans le dispatch de la Gateway (remplacer le stub)

Gateway WF1 `LUMINA-TELEGRAM-GATEWAY` (`uGF3pSv6aTK79cqi`). État live confirmé :
`… → Appel Router → IF clarification →` **TRUE** `Réponse clarification` / **FALSE** `Réponse (stub)`.
On insère la branche mémoire **sur la sortie FALSE de `IF clarification`**, en amont du stub.

### Étape 7 — N6b `IF intent mémoire`
- Ajouter un node **IF** ; le brancher : **`IF clarification` (sortie FALSE) → `IF intent mémoire`**.
- Conditions (type **String**), **combinateur = OR** :
  1. `{{ $json.intent }}` **is equal to** `memory_search`
  2. `{{ $json.intent }}` **is equal to** `general_question`
- (À ce point du flux, `$json` = contrat Router : `intent, brand, query, chat_id, message_id`.)

### Étape 8 — Branche TRUE : `Contrat M3` → `Appel M3` → `Réponse mémoire`
1. **`Contrat M3`** — node **Edit Fields (Set)**, **Keep Only Set Fields = ON**, champs (String) :
   - `query` = `{{ $json.query }}`
   - `brand` = `{{ $json.brand }}`
   - `chat_id` = `{{ $json.chat_id }}`
   - `message_id` = `{{ $json.message_id }}`
2. **`Appel M3`** — node **Execute Sub-workflow** → workflow **`LUMINA-TELEGRAM-MEMORY-SEARCH`**, **Wait for completion = ON** (passe les données entrantes au sous-workflow). Sortie = `{ chat_id, text }`.
3. **`Réponse mémoire`** — node **Telegram → Send Message**, credential **`LUMINA-Lyra - Telegram`** :
   - **Chat ID** = `{{ $json.chat_id }}`
   - **Text** = `{{ $json.text }}`
   - **Options → Append n8n Attribution = OFF** (piège figé).
   - **Options → Parse Mode = None** (le texte peut contenir des `_`/`*` d'extraits → sinon « can't parse entities »).
- Câblage : `IF intent mémoire` (TRUE) → `Contrat M3` → `Appel M3` → `Réponse mémoire`.

### Étape 9 — Branche FALSE : garder le stub
- **`IF intent mémoire` (FALSE) → `Réponse (stub)`** (le node existant, inchangé).
- Les autres intents (`web_search`, `hermes_ops`, `gmail_*`, `unknown`, …) restent en stub jusqu'à M4+.
- **Save** la Gateway (workflow actif ⇒ **Save obligatoire** avant tout re-test externe — piège figé).

---

## 3. Recette — test live depuis Telegram (@luminaLyraBot)

À faire par Karter, dans Telegram, après Save de la Gateway :

1. `/memory lumina c'est quoi Hermes Exec ?` → réponse `🔎 Mémoire (…)` avec 1–3 extraits sourcés (titre + extrait), collection ≠ `episodic`.
2. Question marque **aftrsn** (ex. `/memory aftrsn nos valeurs de marque`) → banque `aftrsn_memory`, extraits `canon`.
3. Question **sans marque claire** → défaut `lumina`, réponse cohérente (pas d'erreur).
4. Question **sans résultat plausible** → « Je n'ai rien trouvé de fiable en mémoire… » (pas d'invention).
5. **Latence** : réponse perçue rapide (limit 5). Ajuster si besoin.
6. **Contrôles** : aucune erreur JSON (body `JSON.stringify`) ; **Sentinel non déclenché** ; aucun « can't parse entities » (Parse Mode None).

**Critères de succès (Karter) : cohérent · fiable · rapide.**

---

## 4. Pièges à respecter (rappels figés)

- **Body HTTP alimenté par expression → TOUJOURS `JSON.stringify`** ; champ `fx` commençant par `{{` (pas `={{`).
- **Code node** : Mode = *Run Once for All Items* car on retourne `[{ json }]`.
- **Après un node qui remplace `$json`** : repropager le contexte via `$('NomDuNode')` (ici `$('Prépare requête')` dans N4).
- **Sortie Telegram** : Attribution OFF + Parse Mode None.
- **Workflow actif ⇒ Save** avant tout re-test externe.
- **Error Workflow = Sentinel** via *Settings*, jamais un node dans le flux.
- **memory-search** reste **non touché** (lecture seule) — on ne fait que l'appeler.

---

## 5. À la validation de M3

Générer et **double-sauvegarder** (projet Claude `Marches-a-suivre/` **+** Google Drive `LUMINA AI DOCS`) :
- `POS-LUMINA_Telegram-Memory-Search_2026-07-XX.md` (**EXACT**) → sous-dossier Drive **POS-EXACT**.
- `POS-GENERIQUE_Branche-memory-search-telegram.md` → sous-dossier Drive **POS-GENERIQUE**.

Chacun avec **Difficultés / Solutions / Lessons learned**. Si structurant : addendum **Bible v1.9** + bump **SPEC-BUILD M3 v0.2** (note d'écart : `limit`/`collection` confirmés sur Build Search ; N4 robuste). **n8n fait foi** — vérifier le live avant d'écrire.

---

*GUIDE-BUILD M3 v0.1 — 2026-07-09. Contrat memory-search figé après inspection directe de Build Search. Boussoles : SPEC-BUILD M3 v0.1 + Telegram v0.2 + Bible v1.8.*
