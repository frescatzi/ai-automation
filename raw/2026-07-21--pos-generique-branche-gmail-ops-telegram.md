---
type: raw
title: "POS-GENERIQUE_Branche-gmail-ops-telegram"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_branche-gmail-ops-telegram.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Brancher une branche « Gmail Ops » (lecture + brouillon) sur une Gateway Telegram

**Type :** patron réutilisable (indépendant des IDs). Dérivé de M5 (After Sun People) mais applicable à toute marque / toute boîte.
**Principe :** un Router classe l'intention ; sur l'intention « email », la Gateway appelle un sous-workflow qui **lit/résume** N boîtes et **prépare des brouillons** — **sans jamais envoyer**. L'envoi reste une action critique gatée (validation séparée).

---

## 1. Quand l'utiliser
Un bot Telegram (ou autre interface) doit consulter une ou plusieurs boîtes Gmail et rédiger des brouillons, sans risque d'envoi automatique. Le Router émet déjà des intents dédiés (`gmail_search` / `gmail_read` / `draft_email` ou équivalents).

## 2. Invariant de sécurité (non négociable)
**Uniquement `Message: Get Many` (lecture) et `Draft: Create` (brouillon). Aucun node d'envoi.** L'envoi = étape ultérieure avec garde-fou de validation. C'est ce qui rend la branche « non critique » et déployable tôt.

## 3. Contrat d'interface
- **Entrée du sous-wf** (depuis la Gateway) : `{ intent, query, brand, chat_id, message_id }`.
- **Sortie du sous-wf** : `{ chat_id, text }` (même forme que les autres branches → le node de réponse Telegram ne change pas).

## 4. Structure du sous-workflow (patron)
```
Receive (Execute Workflow Trigger, Accept all data)
  → Understand request (LLM planificateur → JSON { op:read|draft, mailbox, gmail_query, max, draft:{to,subject,body_points} })
  → Parse plan (Code : normalise + repropage chat_id/message_id/user_request)
  → Route op (Switch read/draft)
     ├─ read  → [Gmail Get Many ×N boîtes] → Tag (Set account=<boîte>) → Merge(append) → Summarize (LLM grounded) → Shape reply (Code)
     └─ draft → Compose draft (LLM → {subject,body}) → Parse draft → Route mailbox (Switch) → Draft: Create ×N → Shape reply (Code)
```

**Réglages clés :**
- LLM planificateur/résumé/rédaction : 1 modèle rapide/économique, **1 passe** chacun (pas d'empilement → latence maîtrisée).
- Gmail Get Many : **Simplify ON** (snippets suffisent, évite le N+1 fetch des corps), **Limit** piloté par le plan, **Search `q`** = opérateurs Gmail traduits par le planificateur, **`alwaysOutputData` ON** (gère la boîte vide → un item vide traverse, le résumé dit « rien trouvé »).
- Draft: Create : Subject/Message/`Options → Send To`, **Operation = Create — jamais Send**.
- Merge = **append**, `numberInputs` = nb de boîtes.
- Sortie Telegram : **Attribution OFF + Parse Mode None** ; crochets `[..]` plutôt que `<..>` (avalés par le HTML Telegram).

## 5. Branchement dans la Gateway
1. **`Is it a <email> question?`** (IF, OR sur l'intent) inséré sur la sortie voulue du dispatch — **et n8n fait foi** : tracer le câblage réel avant d'insérer (la sortie « FALSE » peut déjà pointer ailleurs qu'un stub).
2. TRUE → `Send typing` → `Prepare Input` (Set : repropage le contrat depuis le node Router) → `Run the ops` (Execute Sub-workflow, **Wait = ON**) → `Reply` (Telegram Send).
3. FALSE → **rebrancher vers l'ancienne cible** de la sortie détournée (préserver la chaîne existante) ; supprimer l'ancienne arête.

## 6. Difficultés typiques
1. **Session UI perdue** dans un navigateur piloté → reconnexion nécessaire, puis clics internes only.
2. **Auto-assignation de credential** fiable seulement s'il n'y a **qu'une** credential compatible ; avec plusieurs boîtes → assignation aléatoire à vérifier node par node.
3. **Options de node absentes selon la version** (ex. « Use Responses API » du node OpenAI) — ne pas supposer, vérifier la version installée.
4. **Modal Settings du workflow** parfois non ouvrable en automatisation → Error Workflow + tags à poser à la main.
5. **Tracé/suppression de connexions** au pixel : non fiable ; cliquer une arête ne la sélectionne pas ; Backspace peut supprimer un node resté sélectionné.

## 7. Solutions / méthode robuste
1. **Construire par import JSON** (paste-import via clipboard + Cmd+V, ou `file_upload` sur l'input caché « Import from file ») : une opération courte, résiliente aux coupures. Le code des nodes doit **éviter les backticks littéraux** (regex de fences via ``{3}`) pour rester injectable.
2. **Assigner les credentials via dropdown** : ouvrir → **taper pour filtrer** → cliquer le résultat précisément (le clic sur l'option échoue si mal ciblé ; le filtre réduit l'ambiguïté).
3. **Piloter connexions par le DOM** : lire les `.vue-flow__handle` (`getBoundingClientRect`, convertir à l'échelle capture = capture_px / viewport_px) ; **vérifier chaque arête** après tracé via les `data-id` d'arête. Supprimer via le **milieu réel de courbe** (`getPointAtLength(L/2)` + `getScreenCTM`) puis le `delete-connection-button` du survol. **Vérifier la sélection au DOM avant tout Backspace.**
4. **Réutiliser une branche existante par duplication** (copier/coller) pour hériter des credentials et expressions validées ; ne changer que la cible du sous-wf, les champs spécifiques et les noms.

## 8. Lessons learned (réutilisables)
1. **Vérifier chaque credential** dès qu'il existe >1 candidate.
2. **`$json` clobbering** après un node qui écrit sa propre sortie (ex. Send typing Telegram) → lire le contrat via `$('<node Router>').first().json.<champ>`, pas `$json`.
3. **Set (Edit Fields)** : valeur en **mode Expression** ; l'éditeur auto-ferme `{{ }}` → ne pas retaper les `}}`.
4. **`alwaysOutputData`** sur les lectures pour couvrir le cas « rien trouvé ».
5. **1 passe LLM par étape** (grounding = fact-check) → cohérent, fiable, **rapide** ; l'empilement multiplie la latence sans gain.
6. **n8n fait foi** : tracer le câblage réel avant d'insérer ; ne pas faire confiance aveuglément au guide.
7. **Sécurité by design** : lecture + brouillon uniquement rend la branche non critique et déployable tôt ; l'envoi reste gaté.

---

*POS GÉNÉRIQUE — Branche Gmail Ops (lecture + brouillon) sur Gateway Telegram. Dérivé de M5 LUMINA, 2026-07-13.*
