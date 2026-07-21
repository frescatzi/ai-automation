---
type: raw
title: "POS-GENERIQUE_Router-intention-LLM-n8n"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_router-intention-llm-n8n.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — Router d'intention LLM (n8n) · GÉNÉRIQUE

**Version :** v1.0
**Date :** 2026-07-08
**Projet :** patron réutilisable (dérivé de LUMINA M2 INTENT-ROUTER)
**Étape couverte :** construire un sous-workflow n8n qui classe l'intention d'un message et renvoie un contrat JSON.

> Patron transposable à toute marque `<MARQUE>` / tout bot. Valeurs concrètes remplacées par des paramètres : `<ROUTER_WF>`, `<GATEWAY_CONTRAT>`, `<LLM_MODEL>`, `<ERROR_WF>`, `<TRIGGER_NODE>`.

---

## 1. Objectif

Créer un **sous-workflow classifieur** appelé en `executeWorkflow` par une gateway. Il reçoit un message normalisé, le **classe (intention + marque/scope + drapeaux)** via un LLM, et renvoie un **JSON strict** que la gateway sait dispatcher. Pas d'effet de bord (pas d'appel mémoire, pas d'envoi) : uniquement de la classification.

## 2. Prérequis

1. Une credential LLM (`<LLM_MODEL>`, ex. via OpenRouter) déjà en place.
2. Un contrat d'entrée défini côté gateway `<GATEWAY_CONTRAT>` (au minimum `{ input, mode, command, chat_id, message_id }`).
3. (Optionnel) un workflow d'erreur `<ERROR_WF>` à brancher en Error Workflow.

## 3. Procédure pas à pas (étapes à partir de 1)

### 3.1 — Workflow
1. Créer `<ROUTER_WF>`, tags de rangement (statut + domaine + scope + bot).
2. **Settings → Error Workflow** = `<ERROR_WF>` **si celui-ci est activé** ; sinon laisser vide et le régler avant publication (voir §7.2).
3. Ne pas publier : un sous-workflow appelé en `executeWorkflow` n'a pas besoin d'être actif.

### 3.2 — Node 1 : `<TRIGGER_NODE>` (Execute Workflow Trigger)
1. Ajouter **Execute Workflow Trigger** (« When Executed by Another Workflow »).
2. **Input data mode = Accept all data** (reçoit tout `<GATEWAY_CONTRAT>` sans typage).
3. Lui donner un **nom parlant** (`<TRIGGER_NODE>`) — ce nom sera référencé par le code de l'étape 3.4.

### 3.3 — Node 2 : Classifieur (Basic LLM Chain)
1. Ajouter **Basic LLM Chain**, y rattacher un **Chat Model** (`<LLM_MODEL>`), **température basse (~0.1)**.
2. **Prompt source : Define below**.
3. **User prompt** : projeter les champs utiles du contrat, ex.
   ```
   mode={{ $json.mode }} | command={{ $json.command }} | message={{ $json.input }}
   ```
4. **System message** : imposer une **sortie JSON stricte, sans texte autour**, avec au minimum :
   ```
   { "intent": "<enum d'intentions>", "brand": "<enum de scopes|none>",
     "confidence": 0.0-1.0, "needs_clarification": bool,
     "requires_validation": bool, "query": "<requête reformulée>" }
   Règles :
   - Scope ambigu ET utile → needs_clarification=true, brand="none".
   - requires_validation=true pour toute action critique (envoi, suppression, publication, dépense, modif canon…).
   - Réponses de validation ("valide"/"annule"/"/validate"/"/cancel") → intent="validation_response".
   - Si une commande /… est fournie, elle prime.
   - Ne jamais inventer : incompris → intent="unknown", confidence basse.
   ```

### 3.4 — Node 3 : Parse JSON (Code)
1. Ajouter un node **Code** (JS, Run Once for All Items), nom parlant.
2. **Durcir** la lecture : retirer d'éventuelles fences ```` ``` ````, `JSON.parse` sous `try/catch`, fallback `unknown`, puis **repropager le contexte** (`chat_id`, `message_id`) depuis le trigger via son nom :
   ```javascript
   let raw = ($json.text ?? $json.response ?? '{}').trim();
   raw = raw.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/i, '').trim();
   let out;
   try { out = JSON.parse(raw); }
   catch (e) { out = { intent:'unknown', brand:'none', confidence:0,
     needs_clarification:false, requires_validation:false,
     query: $('<TRIGGER_NODE>').first().json.input ?? '' }; }
   out.brand_available = !['<SCOPES_SANS_BANQUE>'].includes(out.brand);
   out.chat_id    = $('<TRIGGER_NODE>').first().json.chat_id;
   out.message_id = $('<TRIGGER_NODE>').first().json.message_id;
   return [{ json: out }];
   ```

## 4. Vérification / recette

Épingler un input de test dans `<TRIGGER_NODE>`, lancer, lire la sortie du node Parse. Couvrir au moins **3 cas** :
1. **Cas nominal** : message clair avec scope → `intent` attendu, `brand` attendu, contexte repropagé.
2. **Cas clarification** : message utile mais scope ambigu → `needs_clarification=true`, `brand="none"`.
3. **Cas unknown** : charabia → `intent="unknown"`, `confidence` basse.

## 5. Difficultés rencontrées (typiques)

1. Le workflow d'erreur voulu **n'apparaît pas** dans le sélecteur → souvent parce qu'il est **inactif** (grisé) ou porte un **nom complet différent du raccourci** utilisé en doc.
2. Le code de parse référence un node par un libellé provisoire (R1…) qui ne correspond pas au **nom final** du node.
3. Le LLM enrobe son JSON dans des fences → `JSON.parse` échoue.
4. Emplacement du **System Message** variable selon la version du node.

## 6. Solutions implémentées

1. Vérifier le **nom exact + l'ID** du workflow d'erreur ; s'il est inactif, différer son branchement (non bloquant pour un test manuel) ou l'activer.
2. Nommer les nodes **parlant dès le départ** et écrire le code contre ces noms finaux.
3. **Strip des fences** avant `JSON.parse` + fallback `unknown`.
4. Localiser le System Message dans la version courante avant de coller.

## 7. Lessons learned

1. **Toujours consigner nom exact + ID** d'un workflow réutilisé — le raccourci en doc induit en erreur.
2. **Un Error Workflow inactif est grisé** : prévoir son activation au moment de publier la chaîne.
3. **Nommer les nodes de façon parlante** et synchroniser tout `$('Nom')` du code.
4. **Durcir tout parseur de sortie LLM** (strip fences + fallback) : réflexe systématique.
5. **Contrat de sortie stable** (mêmes clés, contexte repropagé) : c'est ce qui permet à la gateway de dispatcher sans connaître le détail du classifieur.

## 8. Références

- Instanciation concrète : `POS-LUMINA_Telegram-Intent-Router_2026-07-08.md`.
- Voisins : `POS-GENERIQUE_Routeur-multi-LLM-par-task_type-passerelle-OpenAI-compatible.md`.
