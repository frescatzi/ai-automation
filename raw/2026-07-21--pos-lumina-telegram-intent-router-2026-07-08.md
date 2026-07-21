---
type: raw
title: "POS-LUMINA_Telegram-Intent-Router_2026-07-08"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_telegram-intent-router_2026-07-08.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — LUMINA-TELEGRAM-INTENT-ROUTER (M2) · EXACT

**Version :** v1.0
**Date :** 2026-07-08
**Projet :** LUMINA OS · Personal Assistant Bot
**Étape couverte :** M2 — INTENT-ROUTER (WF2). SPEC-BUILD v0.2 §6, checklist §9.2.item 2.

---

## 1. Objectif

Construire le **workflow 2** `LUMINA-TELEGRAM-INTENT-ROUTER` : à partir du message normalisé transmis par la Gateway, **classer l'intention + la marque** via un LLM et renvoyer un contrat JSON propre à la Gateway. Aucun accès mémoire, aucun envoi Telegram ici — c'est le cerveau de classification, testé en exécution manuelle.

## 2. Prérequis

1. P1–P4 faits (bot `luminaLyraBot`, credential Telegram `LUMINA-Lyra - Telegram`, auth ACL `lumina_access.bot_acl` + `luminabot` + credential `LUMINA_UserAccess_Postgres`).
2. Credential OpenRouter existante (celle du Maestro), modèle `anthropic/claude-sonnet-latest`.
3. Contrat d'entrée attendu (sortie Gateway, SPEC §5.2) :
   `{ input, mode, command, args, user, chat_id, message_id }`.

## 3. Procédure pas à pas (valeurs réelles)

### 3.1 — Workflow
1. Nouveau workflow, projet **Personal**, nommé exactement **`LUMINA-TELEGRAM-INTENT-ROUTER`**.
   - **Workflow ID réel : `KTLwQi7ZKHDTexMZ`**.
2. Tags posés : **`TEST` · `BRAIN` · `SHARED` · `ASSISTANT-BOT`**.
   - Écart assumé vs SPEC (`🔵 / BRAIN / SHARED / ASSISTANT-BOT`) : `TEST` = équivalent « pas encore publié », `ASSISTANT-BOT` tient le rôle du bot Lyra. Acté tel quel.
3. **Settings → Error Workflow** = **`LUMINA-SENTINEL/ERROR-WATCH`** (branché le 08-07, comme WF1).
4. Non publié (WF2 est appelé en `executeWorkflow`).

### 3.2 — Node 1 : `Réception Gateway` (Execute Workflow Trigger)
- Type : `n8n-nodes-base.executeWorkflowTrigger` (« When Executed by Another Workflow »).
- **Input data mode : Accept all data**.
- Renommé **`Réception Gateway`** (nom parlant ; c'est le node référencé par le code de l'étape 3.4).

### 3.3 — Node 2 : `Classifieur d'intention` (Basic LLM Chain)
- Type : `@n8n/n8n-nodes-langchain.chainLlm`, renommé **`Classifieur d'intention`**.
- Modèle rattaché : **OpenRouter Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`), credential OpenRouter du Maestro, modèle `anthropic/claude-sonnet-latest`, **Temperature = 0.1**.
- **Prompt source : Define below**.
- **User prompt** :
  ```
  mode={{ $json.mode }} | command={{ $json.command }} | message={{ $json.input }}
  ```
- **System message** :
  ```
  Tu es le classifieur d'intention du LUMINA Assistant Bot (usage privé de Karter).
  À partir du message, renvoie UNIQUEMENT un JSON valide, sans texte autour :
  {
    "intent": "general_question|web_search|memory_search|gmail_search|gmail_read|draft_email|calendar_read|calendar_action|save_note|hermes_ops|validation_response|unknown",
    "brand": "lumina|aftrsn|karter|agency|none",
    "confidence": 0.0-1.0,
    "needs_clarification": true|false,
    "requires_validation": true|false,
    "query": "la requête utile reformulée"
  }
  Règles :
  - Marque détectée par mots-clés/contexte ; si ambiguë ET utile → needs_clarification=true, brand="none".
  - requires_validation=true pour toute action critique (envoi email, suppression, publication, dépense, modif workflow, modif mémoire canon, événement avec invités).
  - Les réponses "oui valide"/"non annule"/"/validate"/"/cancel" → intent="validation_response".
  - Si une commande /… est fournie, elle prime pour déterminer l'intent.
  - Ne jamais inventer : si tu ne comprends pas → intent="unknown", confidence basse.
  ```

### 3.4 — Node 3 : `Parse JSON sortie` (Code)
- Type : `n8n-nodes-base.code`, JavaScript, **Run Once for All Items**, renommé **`Parse JSON sortie`**.
- Code (durci : retire un éventuel bloc ```` ```json ```` avant parse ; référence le trigger par son nom réel `Réception Gateway`) :
  ```javascript
  let raw = ($json.text ?? $json.response ?? '{}').trim();
  raw = raw.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/i, '').trim();

  let out;
  try { out = JSON.parse(raw); }
  catch (e) {
    out = { intent:'unknown', brand:'none', confidence:0,
            needs_clarification:false, requires_validation:false,
            query: $('Réception Gateway').first().json.input ?? '' };
  }
  out.brand_available = !['karter','agency'].includes(out.brand);
  out.chat_id    = $('Réception Gateway').first().json.chat_id;
  out.message_id = $('Réception Gateway').first().json.message_id;
  return [{ json: out }];
  ```
- Flux final : `Réception Gateway → Classifieur d'intention → Parse JSON sortie`. Sauvegardé (Cmd+S).

## 4. Vérification / recette (exécution manuelle, 08-07)

Méthode : épingler l'input dans `Réception Gateway`, **Test workflow**, lire la sortie de `Parse JSON sortie`.

| Test | Input (extrait) | Sortie obtenue | Verdict |
|---|---|---|---|
| **T4** | `/memory lumina c'est quoi Hermes Exec ?` (mode command) | `intent=memory_search, brand=lumina, confidence=0.75, brand_available=true, chat_id=776345147, message_id=1` | ✅ conforme SPEC §8 T4 |
| **Cas A** (clarification) | « cherche dans la base ce qu'on a décidé » (natural) | `brand=none, needs_clarification=true` (intent=memory_search, confidence=0.55) | ✅ |
| **Cas B** (unknown) | « asdkfj qwpoe azer » (natural) | `intent=unknown, confidence=0.05` | ✅ (avec `needs_clarification=true` en plus — mineur) |

DoD M2 : R1→R2→R3 tourne en manuel et sort le contrat §6.3 attendu. **Atteint.**

## 5. Difficultés rencontrées

1. **Error Workflow « Sentinel » introuvable** dans le sélecteur — perçu comme « Sentinel n'existe pas ». En réalité il apparaissait **grisé avec ⚠️** sous son nom complet `LUMINA-SENTINEL/ERROR-WATCH`. Cause : le workflow est **inactif**, et n8n grise les workflows inactifs dans le sélecteur d'Error Workflow.
2. **Nommage des nodes** : le SPEC utilisait des libellés R1/R2/R3 ; on a préféré des noms parlants, ce qui casse la référence `$('R1')` du code fourni.
3. **Piège JSON en fences** : le classifieur peut enrober sa réponse dans ```` ```json … ``` ````, ce qui ferait échouer un `JSON.parse` brut → faux `unknown`.
4. **Emplacement du System Message** variable selon la version de n8n du Basic LLM Chain.

## 6. Solutions implémentées

1. Sentinel identifié = `LUMINA-SENTINEL/ERROR-WATCH` (`xzaH0uWy0idKVphF`), état « non activé » déjà fiché en Bible. Décision : **laisser `- No Workflow -`** pour le build/test manuel (non bloquant) et **régler l'Error Workflow avant de publier la Gateway** (activer Sentinel ou l'accepter grisé selon le comportement n8n). Ajouté en reste-à-faire.
2. Noms parlants adoptés : `Réception Gateway`, `Classifieur d'intention`, `Parse JSON sortie` ; le code de R3 a été **réécrit pour référencer `$('Réception Gateway')`**.
3. R3 **durci** : `raw.replace(/^```(?:json)?\s*/i,'').replace(/\s*```$/i,'')` avant `JSON.parse`, plus fallback `unknown` propre.
4. Prompts collés une fois le champ System Message localisé ; procédure « dis-moi ce que tu vois » retenue pour les écarts d'UI.

## 7. Lessons learned

1. **Nom raccourci ≠ nom réel n8n.** « Sentinel » = `LUMINA-SENTINEL/ERROR-WATCH`. Toujours consigner **nom exact + ID** dans les docs pour éviter le « ça n'existe pas ». (Remonter : garder cette convention dans la Bible.)
2. **Error Workflow inactif = grisé** dans le sélecteur. Pour le brancher réellement, prévoir de **l'activer** au moment de publier — à trancher pour la Gateway.
3. **Nommer les nodes parlant dès le départ**, et synchroniser tout code qui référence un node par `$('Nom')`. Un guide de build doit livrer le code déjà aligné sur les noms finaux.
4. **Durcir systématiquement le parse de sortie LLM** (strip fences + fallback) : évite les faux `unknown` — piège figé, à réappliquer sur tous les futurs parseurs de classifieur.
5. **Watch-point classifieur** : sur du charabia il met `needs_clarification=true`. Sans conséquence ici (Gateway route `unknown → /help`), mais à surveiller si un jour la clarification devient coûteuse.

## 8. Références

- SPEC-BUILD v0.2 §6 (nodes, prompts, contrat §6.3), §8 (T4), §9.2.item 2.
- Bible 360° : fiche `LUMINA-SENTINEL/ERROR-WATCH` (`xzaH0uWy0idKVphF`, « non activé »).
- Workflow réel : `LUMINA-TELEGRAM-INTENT-ROUTER` — ID **`KTLwQi7ZKHDTexMZ`** (projet Personal).
- OpenRouter `anthropic/claude-sonnet-latest`, température 0.1.
- Générique associé : `POS-GENERIQUE_Router-intention-LLM-n8n.md`.
