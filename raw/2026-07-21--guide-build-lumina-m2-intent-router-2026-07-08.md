---
type: raw
title: "GUIDE-BUILD_LUMINA-M2-Intent-Router_2026-07-08"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m2-intent-router_2026-07-08.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# 🔧 GUIDE DE BUILD — M2 · LUMINA-TELEGRAM-INTENT-ROUTER (WF2)

**Version :** v1.0 · **Date :** 2026-07-08 · **Projet :** LUMINA OS · Personal Assistant Bot
**Étape couverte :** M2 (INTENT-ROUTER) — SPEC-BUILD v0.2 §6, checklist §9.2
**Nature :** guide de construction *à exécuter dans n8n par Karter*. Ce n'est pas encore la POS finale — la POS exacte + générique (avec difficultés / solutions / lessons) est produite **après ta validation**, puis double-sauvegardée (projet Claude + Drive).

> Boussole : SPEC-BUILD Telegram v0.2 §6 + Bible 360° v1.4. **n8n fait foi** : si un node ne se comporte pas comme décrit, on vérifie le live et on m'en parle avant d'avancer.

---

## 1. Objectif

Créer le **workflow 2** qui, à partir du message normalisé envoyé par la Gateway, **classe l'intention + la marque** et renvoie un contrat JSON propre. À la fin de M2 : R1 → R2 → R3 tourne en **exécution manuelle** avec un input factice et sort le JSON attendu (§5 ci-dessous). Pas de Telegram, pas de Gateway ici — juste le cerveau de classification.

## 2. Prérequis

1. P1–P4 faits (bot `luminaLyraBot`, auth ACL) — OK au 08-07.
2. Credential OpenRouter déjà en place (celle utilisée par le Maestro, modèle `~anthropic/claude-sonnet-latest`).
3. Error Workflow Sentinel connu : `xzaH0uWy0idKVphF`.
4. Contrat d'entrée attendu (sortie Gateway, SPEC §5.2) :
   ```json
   { "input":"…", "mode":"command|natural", "command":"/memory|null",
     "args":"…", "user":"karter", "chat_id":"…", "message_id":"…" }
   ```

## 3. Procédure pas à pas (dans n8n)

### 3.1 — Créer le workflow

1. **New workflow** → nom exact : `LUMINA-TELEGRAM-INTENT-ROUTER`.
2. Tags : `🔵` (à publier) · `🤖 BRAIN` · `🌐 SHARED` · `ASSISTANT-BOT`.
3. **Settings → Error Workflow** = `Sentinel` (`xzaH0uWy0idKVphF`).
4. *Ne pas publier* : WF2 est appelé en `executeWorkflow` par la Gateway, il n'a pas besoin d'être actif.

### 3.2 — R1 · Execute Workflow Trigger

1. Ajouter le node **Execute Workflow Trigger** (`n8n-nodes-base.executeWorkflowTrigger`).
2. **Input data mode** : *Accept all data* (le plus simple pour le MVP ; on récupère tout le contrat §2 tel quel). Si tu préfères typer les champs, définis : `input, mode, command, args, user, chat_id, message_id`.
3. C'est le point d'entrée : il reçoit le contrat §5.2 de la Gateway.

### 3.3 — R2 · Classifier (LLM)

1. Ajouter un node **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`), branché après R1.
2. Y rattacher un **OpenRouter Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) :
   - Modèle : `~anthropic/claude-sonnet-latest` (même que le Maestro).
   - **Température : ~0.1** (classification déterministe).
3. **System prompt** (coller tel quel) :
   ```text
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
4. **User prompt** :
   ```text
   mode={{ $json.mode }} | command={{ $json.command }} | message={{ $json.input }}
   ```
5. ⚠️ **Piège à surveiller** : certains modèles enveloppent le JSON dans des balises ```` ```json ````. R3 (ci-dessous) est durci pour absorber ce cas — mais si tu vois `intent:"unknown"` de façon systématique au test, c'est presque toujours ça : regarde la sortie brute de R2.

### 3.4 — R3 · Parse JSON (Code)

1. Ajouter un node **Code** (`n8n-nodes-base.code`), branché après R2.
2. Coller (version durcie : retire les fences ```` ``` ```` éventuelles avant `JSON.parse`, comportement identique au SPEC sinon) :
   ```javascript
   // récupère le texte du LLM (chainLlm sort .text, parfois .response)
   let raw = ($json.text ?? $json.response ?? '{}').trim();
   // durcissement : retire un éventuel bloc ```json ... ```
   raw = raw.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/i, '').trim();

   let out;
   try { out = JSON.parse(raw); }
   catch (e) {
     out = { intent:'unknown', brand:'none', confidence:0,
             needs_clarification:false, requires_validation:false,
             query: $('R1').first().json.input ?? '' };
   }
   // garde-fous marque MVP : karter/agency pas encore de banque
   out.brand_available = !['karter','agency'].includes(out.brand);
   // repropage le contexte utile pour la réponse Gateway
   out.chat_id    = $('R1').first().json.chat_id;
   out.message_id = $('R1').first().json.message_id;
   return [{ json: out }];
   ```
3. **Attention aux noms de nodes** dans `$('R1')` : si tu n'as pas renommé le trigger en `R1`, remplace par son vrai nom (ou renomme le node `R1` pour coller au guide). Idem `R2`/`R3` sont des libellés de repère — le seul nom référencé en code est celui du **trigger**.

## 4. Vérification / recette (exécution manuelle)

1. Sur le node **R1**, ouvrir **Execute step / pin data** et coller cet input factice :
   ```json
   { "input":"/memory lumina c'est quoi Hermes Exec ?", "mode":"command",
     "command":"/memory", "args":"lumina c'est quoi Hermes Exec ?",
     "user":"karter", "chat_id":"776345147", "message_id":"1" }
   ```
2. **Test workflow** (exécution complète R1→R2→R3).
3. **Sortie attendue en R3** (contrat §6.3) :
   ```json
   { "intent":"memory_search", "brand":"lumina", "confidence": 0.8,
     "needs_clarification": false, "requires_validation": false,
     "query":"c'est quoi Hermes Exec", "brand_available": true,
     "chat_id":"776345147", "message_id":"1" }
   ```
   (les valeurs `confidence`/`query` peuvent varier légèrement ; l'essentiel : `intent=memory_search`, `brand=lumina`, `brand_available=true`, `chat_id`/`message_id` repropagés.)
4. **Deux tests de robustesse rapides** avant de dire OK :
   - Input `{ "input":"cherche dans la base ce qu'on a décidé", "mode":"natural", "command":null, "chat_id":"776345147", "message_id":"2" }` → attendu `needs_clarification=true`, `brand="none"`.
   - Input `{ "input":"asdkfj qwpoe", "mode":"natural", "command":null, "chat_id":"776345147", "message_id":"3" }` → attendu `intent="unknown"`, `confidence` basse.

## 5. Quand c'est validé

Dès que tu me confirmes que M2 sort le bon JSON (§4), je génère automatiquement et je **double-sauvegarde** (projet Claude `Marches-a-suivre/` + Drive `LUMINA AI DOCS`, sous-dossiers POS-EXACT / POS-GENERIQUE) :
- `POS-LUMINA_Telegram-Intent-Router_2026-07-08.md` (exact)
- `POS-GENERIQUE_Router-intention-LLM-n8n.md`

…chacun avec **Difficultés rencontrées / Solutions implémentées / Lessons learned**, et je mets à jour les boussoles si l'étape est structurante (Bible addendum + bump, SPEC-BUILD note d'écart). Puis on enchaîne **M1 — GATEWAY**.

## 6. Références

- SPEC-BUILD v0.2 §6 (nodes R1–R3, prompts, contrats §6.3), §9.2 (checklist), §3 (IDs briques).
- Bible 360° v1.4 (addendum Assistant Bot).
- Sentinel error workflow : `xzaH0uWy0idKVphF`. OpenRouter `~anthropic/claude-sonnet-latest`.
