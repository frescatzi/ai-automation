---
type: raw
title: "POS-LUMINA_Telegram-Gateway_2026-07-08"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_telegram-gateway_2026-07-08.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — LUMINA-TELEGRAM-GATEWAY (M1) · EXACT

**Version :** v1.0
**Date :** 2026-07-08
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra »
**Étape couverte :** M1 — GATEWAY (WF1). SPEC-BUILD v0.2 §5, checklist §9 items 3–7.

---

## 1. Objectif

Construire le **workflow 1** `LUMINA-TELEGRAM-GATEWAY` : recevoir un message Telegram de Lyra (@luminaLyraBot), **authentifier** l'expéditeur (ACL Postgres), traiter `/help`, sinon **appeler le Router (WF2)** et **répondre** (réel pour `/help` + clarification, stub pour les intentions dont le module n'existe pas encore). À la fin : un message réel de Karter à Lyra déclenche une réponse de bout en bout.

## 2. Prérequis

1. WF2 `LUMINA-TELEGRAM-INTENT-ROUTER` (ID `KTLwQi7ZKHDTexMZ`) construit et validé (M2).
2. Bot **Lyra** @luminaLyraBot ; credential Telegram **`LUMINA-Lyra - Telegram`** (token de Lyra).
3. Auth ACL : base `lumina_access`, table `bot_acl`, credential **`LUMINA_UserAccess_Postgres`** ; owner `chat_id` **776345147** = `role owner`.

## 3. Procédure pas à pas (valeurs réelles)

### 3.1 — Workflow
1. Nouveau workflow, projet **Personal**, nommé **`LUMINA-TELEGRAM-GATEWAY`** (projet Personal). **ID réel : `uGF3pSv6aTK79cqi`.**
2. Tags : `TEST · BRAIN · SHARED · ASSISTANT-BOT`.
3. **Settings → Error Workflow** = **`LUMINA-SENTINEL/ERROR-WATCH`** — branché le 08-07 sur **WF1 et WF2** (une fois Sentinel devenu sélectionnable). Filet d'erreurs global via le mécanisme Error Workflow, **pas** un node dans le flux.

### 3.2 — Nodes (flux linéaire + branches)

**N1 `Réception Telegram`** — Telegram Trigger, *Trigger On = Message*, **credential `LUMINA-Lyra - Telegram`**.
> ⚠️ Credential = **le bot que l'utilisateur contacte** (Lyra). Ne PAS mettre `LuminaOsBot` (alertes) : un seul trigger par bot, et ce serait le mauvais bot (cf. §5.4).

**N2 `Normalize`** — Code, **Mode = Run Once for All Items**, retourne `[{ json: {...} }]` :
```javascript
const m = $json.message ?? {};
const chatId   = String(m.chat?.id ?? '');
const text     = (m.text ?? '').trim();
const isCommand = text.startsWith('/');
const command   = isCommand ? text.split(/\s+/)[0].toLowerCase() : null;
const args      = isCommand ? text.slice(command.length).trim() : text;
return [{ json: {
  chat_id: chatId, message_id: m.message_id ?? null, input: text,
  mode: isCommand ? 'command' : 'natural', command, args
}}];
```

**N2b `Auth. Access Control List`** — Postgres (node v2.6), *Execute Query*, credential `LUMINA_UserAccess_Postgres` :
- Query (toujours 1 ligne, `role` = null si non autorisé) :
  ```sql
  SELECT (SELECT role FROM bot_acl WHERE chat_id = $1) AS role;
  ```
- Query Parameters : `{{ $json.chat_id }}`
- Settings → **On Error = Stop Workflow** (fail-safe).

**N3 `IF autorisé`** — IF : `{{ $json.role }}` **is not empty (String)**.
- **FALSE → `Accès refusé`** : Telegram Send Message, credential `LUMINA-Lyra - Telegram`, Chat ID `{{ $('Normalize').item.json.chat_id }}`, texte `🚫 Access denied`.

**N4 `IF /help`** — sur TRUE de N3. IF : `{{ $('Normalize').item.json.command }}` **is equal to** `/help`.
- **TRUE → `Réponse /help`** : Telegram Send Message, Chat ID `{{ $('Normalize').item.json.chat_id }}`, texte = liste des commandes de Lyra.

**N5a `Contrat Router`** — sur FALSE de N4. Edit Fields (Set), **Keep Only Set Fields**, champs (String) depuis `$('Normalize')` : `input, mode, command, args, user='karter', chat_id, message_id`.

**N5 `Appel Router`** — Execute Sub-workflow → **`LUMINA-TELEGRAM-INTENT-ROUTER`** (`KTLwQi7ZKHDTexMZ`), **Wait for completion = ON**. Sortie = contrat Router (`intent, brand, …, chat_id, message_id`).

**N6 `IF clarification`** — IF : `{{ $json.needs_clarification }}` **is true (Boolean)**.
- **TRUE → `Réponse clarification`** : Telegram Send Message, Chat ID `{{ $json.chat_id }}`, texte « Dans quelle base veux-tu chercher : LUMINA ou AFTRSN ? … ».
- **FALSE → `Réponse (stub)`** : Telegram Send Message, Chat ID `{{ $json.chat_id }}`, texte (Expression) `🧭 Intention détectée : {{ $json.intent }} (marque : {{ $json.brand }}). Module en construction. ✨`.

### 3.3 — Mise en service
1. Sur `Réception Telegram`, **dépingler** les données de test.
2. **Save** + **Active** (le toggle production enregistre le webhook Telegram sur Lyra).
3. Vérifier `https://api.telegram.org/bot<TOKEN_LYRA>/getWebhookInfo` → **`url` non vide** = webhook posé.

## 4. Vérification / recette (live Telegram, 08-07)

Messages envoyés à @luminaLyraBot :

| Envoi | Réponse | Verdict |
|---|---|---|
| `/help` | liste des commandes | ✅ |
| `/memory lumina c'est quoi Hermes Exec ?` | 🧭 Intention détectée : memory_search (marque : lumina)… | ✅ |
| `cherche dans la base ce qu'on a décidé` | « Dans quelle base… LUMINA ou AFTRSN ? » | ✅ |
| `asdkfj qwpoe` | 🧭 Intention détectée : unknown… | ✅ |

T2 (refus depuis un autre compte) : **non testé** (pas de 2ᵉ compte Telegram) — logique en place (branche FALSE `IF autorisé` → `Accès refusé`).

## 5. Difficultés rencontrées

1. **Code node — « A 'json' property isn't an object »** : `Normalize` en *Run Once for Each Item* alors que le code retourne un tableau `[{json}]`.
2. **Postgres v2.6** : « Continue On Fail » disparu des Options → désormais **Settings → On Error**. Et une requête `SELECT role FROM bot_acl WHERE chat_id=$1` renvoie **0 ligne** si non autorisé → l'item est perdu, pas de réponse possible.
3. **Telegram « Bad Request: chat not found »** à l'envoi : le nouveau bot Lyra n'avait **aucune conversation ouverte** avec l'owner.
4. **Aucune réaction aux messages live** : `getWebhookInfo` → `url:""`. Cause racine : le trigger `Réception Telegram` pointait sur la credential **`LuminaOsBot`** (bot des alertes), pas **`LUMINA-Lyra`** → le mauvais bot écoutait. De plus, le webhook prod ne s'enregistre qu'à l'**activation**, et on ne peut pas « Test-listen » pendant que le workflow est actif.
5. **Contexte perdu après Auth** : après le node Postgres, `$json = { role }` seulement — plus de `chat_id`/`command`.
6. **Sortie Telegram** : (a) signature automatique « This message was sent automatically with n8n » sur chaque message ; (b) `Bad Request: can't parse entities` sur `web_search` — le `_` des valeurs d'intent (`web_search`, `memory_search`) est lu comme un italique Markdown non fermé.

## 6. Solutions implémentées

1. `Normalize` → **Mode = Run Once for All Items** (accord avec le `return [{json}]`).
2. Requête **toujours 1 ligne** : `SELECT (SELECT role FROM bot_acl WHERE chat_id=$1) AS role;` → `role=null` si non autorisé, item conservé. **On Error = Stop Workflow** = fail-safe (aucun accès accordé si la base tombe).
3. Ouvrir Telegram → **/start** sur @luminaLyraBot avant que Lyra puisse écrire.
4. Trigger recâblé sur **`LUMINA-Lyra - Telegram`** ; workflow passé **Active** (webhook posé, `url` rempli) ; ne pas cliquer « Test this trigger » en mode actif.
5. En aval de l'Auth, lire les champs message via **`$('Normalize')`** ; le contrat pour le Router est reconstruit dans `Contrat Router`.
6. Sur **chaque node d'envoi Telegram** : **Append n8n Attribution = OFF** ; **Parse Mode = None** (ou **HTML**, qui n'interprète pas `_`) car les réponses affichent des valeurs d'intent avec underscores. `/help` réécrit en texte propre (puces, crochets `[..]` au lieu de `<..>` avalés par le HTML).

## 7. Lessons learned

1. **La credential du Telegram Trigger = le bot que l'utilisateur contacte.** Un seul trigger par bot ; ne jamais réutiliser le bot d'alertes (`LuminaOsBot`) pour l'interface. *(Piège figé, à remonter Bible.)*
2. **Le webhook de prod ne s'enregistre qu'à l'activation** ; vérifier `getWebhookInfo` (`url` non vide, `pending_update_count`).
3. **Nouveau bot = /start obligatoire** côté user avant tout envoi (sinon « chat not found »).
4. **Code node : le Mode doit matcher la forme de retour** (All Items → `[{json}]` ; Each Item → objet seul).
5. **Auth au niveau data** : requête ACL toujours-1-ligne + `On Error=Stop` = refus par défaut, jamais d'ouverture accidentelle.
6. **Après un node qui remplace `$json`** (Postgres), repropager le contexte via `$('NomDuNode')`.
7. **Réglages sortie Telegram à figer** : désactiver l'attribution n8n ; éviter le Parse Mode Markdown si le texte contient des `_` dynamiques (valeurs d'enum) → None ou HTML. Piège figé, à réappliquer sur tous les futurs nodes de réponse (M3+).

## 8. Références

- SPEC-BUILD v0.2 §5 (nodes N1–N7), §8 (recette T1–T8), §9 items 3–7.
- WF1 `LUMINA-TELEGRAM-GATEWAY` (ID `uGF3pSv6aTK79cqi`, projet Personal) → WF2 `LUMINA-TELEGRAM-INTENT-ROUTER` (`KTLwQi7ZKHDTexMZ`).
- Auth : `lumina_access.bot_acl` / `luminabot` / `LUMINA_UserAccess_Postgres`. Bot Lyra @luminaLyraBot / `LUMINA-Lyra - Telegram`.
- Générique associé : `POS-GENERIQUE_Gateway-Telegram-auth-dispatch-n8n.md`.
- Reste-à-faire : tester T2 (refus depuis un autre compte) ; rotation mdp `luminabot`. *(Error Workflow `LUMINA-SENTINEL/ERROR-WATCH` branché sur WF1+WF2 le 08-07.)*
