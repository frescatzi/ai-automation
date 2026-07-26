---
type: raw
title: "POS-LUMINA_Telegram-Gmail-Ops_2026-07-13"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_telegram-gmail-ops_2026-07-13.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS — LUMINA-TELEGRAM · M5 GMAIL OPS (EXACT)

**Version :** v1.0
**Date :** 2026-07-13
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra » (@luminaLyraBot)
**Étape couverte :** M5 — GMAIL OPS. Brancher les intents `gmail_search` / `gmail_read` / `draft_email` sur un sous-workflow Gmail dédié : **lire / résumer** sur les deux boîtes business After Sun People, et **préparer des brouillons** — **jamais d'envoi** (l'envoi = M8).
**Boussoles :** GUIDE-BUILD M5 v1.2 · SPEC-BUILD Telegram v0.3 · GUIDE-BUILD M4 v1.0 · Bible 360° v2.5. **n8n fait foi.**

---

## 1. Objectif

Quand le Router classe un message en `gmail_search` / `gmail_read` / `draft_email`, la Gateway appelle un sous-workflow dédié qui : comprend la demande (planificateur LLM), interroge Gmail sur les **deux boîtes** (B2C `hello@` + B2B `partners@`), puis **résume** (regroupé par boîte) **ou** **crée un brouillon** — et renvoie `{ chat_id, text }` dans Telegram.

**Invariant de sécurité (Bible J.2) :** en M5 on n'utilise **que** `Message: Get Many` (lecture) et `Draft: Create` (brouillon). **Aucun node d'envoi.** L'envoi passera par la validation M8.

**Validé live le 2026-07-13** : T1 résumé B2C/B2B ✅ · T2 `is:unread` ✅ · T3 brouillon réellement créé dans Brouillons `hello@` ✅ · T5 zéro envoi (garanti par construction) ✅ · T6 non-régression memory/web/Maestro/clarification ✅ · T7 sorties propres ✅.

## 2. Prérequis (acquis)

1. **Gateway WF1** `LUMINA-TELEGRAM-GATEWAY` — **`uGF3pSv6aTK79cqi`** (Personal, Publié).
2. **Router WF2** `LUMINA-TELEGRAM-INTENT-ROUTER` — **`KTLwQi7ZKHDTexMZ`** : l'enum du classifieur émet bien `gmail_search`, `gmail_read`, `draft_email` (vérifié en live, node « Classifieur d'intention »). Sortie `{ intent, brand, confidence, needs_clarification, requires_validation, query, chat_id, message_id }`. Note : `requires_validation=true` pour toute action critique (**envoi** email, suppression…) — un brouillon n'en est pas une.
3. **Sentinel** `xzaH0uWy0idKVphF` (Error Workflow).
4. Credentials Gmail OAuth2 (créées par Karter) : **`HELLO-AFTRSN GMAIL`** (B2C `hello@aftersunpeople.com`) + **`PARTNERS-AFTRSN GMAIL`** (B2B `partners@aftersunpeople.com`). Credential LLM : **`OpenAi account`**. Credentials Telegram : **`LuminaOsBot - Telegram`** (Send Chat Action) + **`LUMINA-Lyra - Telegram`** (Send Message) — **ce sont deux credentials distinctes**, à mirrorer exactement (repris de la branche web/hermès).

## 3. Sous-workflow `LUMINA-TELEGRAM-GMAIL-OPS`

**Workflow réel :** ID **`LUCkYnz7GhjkyCtZ`** · projet **Personal** · **Publié**. *(Publié sans conséquence externe : déclenché par Execute Workflow, pas de webhook.)*
**Reste à faire (manuel) :** Settings → **Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH` (`xzaH0uWy0idKVphF`)** + tags `TEST→PROD · BRAIN · SHARED · ASSISTANT-BOT` (le modal Settings ne s'ouvrait pas via l'automatisation navigateur — cf. §7).

> Piège figé confirmé : ce sous-wf a un **Execute Workflow Trigger** → **invisible en MCP** (le MCP n'expose que les wf publiés à trigger Schedule/Webhook/Form/Chat). Inspection par canvas/JSON uniquement.

**Flux :**
```
Receive from Gateway → Understand request (LLM) → Parse plan → Route op (Switch)
  ├─ read  → [Gmail B2C + Gmail B2B] → tags → Merge mailboxes → Summarize (LLM) → Shape reply (read)
  └─ draft → Compose draft (LLM) → Parse draft → Route mailbox (Switch) → Create Draft B2C/B2B → Shape reply (draft)
```

### N1 — `Receive from Gateway`
- **Execute Workflow Trigger** (`executeWorkflowTrigger`), **Input Source = Accept all data** (`inputSource: passthrough`). Reçoit `{ intent, query, brand, chat_id, message_id }`.

### N2 — `Understand request` (planificateur)
- **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`), **Source for Prompt = Define below**. Prompt (User Message) = instructions planificateur + `\n\nDEMANDE: intent={{ $json.intent }} | message={{ $json.query }}`. *(Choix : tout dans le prompt utilisateur, pas de message système séparé → robustesse à l'import, comportement identique pour un planificateur mono-tour.)* Renvoie UNIQUEMENT un JSON `{ op, mailbox, gmail_query, max, draft:{to,subject,body_points} }`.
- **Sous-node** : `Model - plan` = **OpenAI Chat Model** (`lmChatOpenAi`), credential **`OpenAi account`**, **Model = `gpt-5-mini`** (From list). ⚠️ **Pas de toggle « Use Responses API »** dans cette version du node → API standard ; **gpt-5-mini fonctionne parfaitement ainsi** (validé T1).

### N3 — `Parse plan` (Code, Run Once for All Items)
Normalise le plan (op read/draft, mailbox b2c/b2b/both, max 1–20 déf. 10, gmail_query déf. `newer_than:2d`), repropage `user_request`, `chat_id`, `message_id`. Lit le contexte via `$('Receive from Gateway').first()`. Strip des fences ```` ```json ```` via regex ``{3}` (évite un backtick littéral dans le code).

### N4 — `Route op` (Switch v3.2 sur `{{ $json.op }}`)
- Sortie 0 **`read`** → branche lecture · Sortie 1 **`draft`** → branche brouillon.

### Branche LECTURE
- **N5a `Gmail B2C`** (`gmail` v2.1, **Message / Get Many**), credential **`HELLO-AFTRSN GMAIL`** · Return All OFF · **Limit** `{{ $('Parse plan').first().json.max }}` · **Simplify ON** · **Filters → Search (`q`)** `{{ $('Parse plan').first().json.gmail_query }}` · **`alwaysOutputData` = ON** (gère la boîte vide, T4).
- **N5a' `Tag b2c`** (Set v3.4, Include Other Fields ON) : `account = b2c`.
- **N5b `Gmail B2B`** : idem, credential **`PARTNERS-AFTRSN GMAIL`**, `alwaysOutputData` ON.
- **N5b' `Tag b2b`** : `account = b2b`.
- **N6 `Merge mailboxes`** (`merge` v3, mode **append**, `numberInputs: 2`) ← Tag b2c (input 0) + Tag b2b (input 1).
- **N7 `Summarize`** (chainLlm) : prompt grounding strict, regroupe B2C/B2B, `DEMANDE = {{ $('Parse plan').first().json.user_request }}`, `EMAILS = {{ JSON.stringify($input.all().map(i => i.json)) }}`. Sous-node `Model - summary` (gpt-5-mini, `OpenAi account`).
- **N8 `Shape reply (read)`** (Code) : `{ chat_id, text }`, `text` tronqué à 3800 (marge sous la limite Telegram 4096). Lit `$('Parse plan').first()`.

### Branche BROUILLON (jamais d'envoi)
- **N9 `Compose draft`** (chainLlm) : rédige un brouillon FR, renvoie `{subject, body}`. Sous-node `Model - draft` (gpt-5-mini, `OpenAi account`). Lit `$('Parse plan').first().json.draft.*`.
- **N10 `Parse draft`** (Code) : parse `{subject, body}`, repropage `to`, `mailbox`, `chat_id`.
- **N11 `Route mailbox`** (Switch v3.2 sur `{{ $json.mailbox }}`) → 0 `b2c` / 1 `b2b`.
- **N12a `Create Draft B2C`** (`gmail` v2.1, **Draft / Create**), credential **`HELLO-AFTRSN GMAIL`** : Subject `{{ $json.subject }}` · Message `{{ $json.body }}` · **Options → Send To** `{{ $json.to }}`. ⚠️ **Operation = Create — jamais Send.**
- **N12b `Create Draft B2B`** : idem, credential **`PARTNERS-AFTRSN GMAIL`**.
- **N13 `Shape reply (draft)`** (Code) : message Telegram « [Brouillon créé dans la boîte … a …] (jamais envoyé) … A relire / valider avant tout envoi (l envoi passera par la validation M8). ». Lit `$('Parse draft').first()`.

Sortie du sous-workflow (branche exécutée) = **`{ chat_id, text }`**.

## 4. Branchement Gateway (dispatch)

Câblage réel (n8n fait foi — différent du guide, qui ignorait « Is it Maestro? ») :
`Classify the intent → Need clarification? →(FALSE)→ Question for Memory-Search? →(FALSE)→ Is it a web/ops question?`

Insertion M5 sur la **sortie FALSE de `Is it a web/ops question?`** (`68a28621…`, out 1), qui pointait vers `Is it Maestro?` :

1. **`Is it a Gmail question?`** (IF v2.2, conditions **String**, combinateur **OR**) : `{{ $('Classify the intent').first().json.intent }}` = `gmail_search` **OU** `gmail_read` **OU** `draft_email`. Entrée = sortie **FALSE** de `Is it a web/ops question?`.
2. Branche **TRUE** :
   - **`Send typing (gmail)`** (Telegram **Send Chat Action**, credential **`LuminaOsBot - Telegram`**) : Chat ID `{{ $('Classify the intent').first().json.chat_id }}`, Action Typing.
   - **`Prepare Input for Gmail`** (Set v3.4 Manual, Include Other Fields OFF) : `intent`, `query`, `brand`, `chat_id`, `message_id`, chacun `{{ $('Classify the intent').first().json.<champ> }}`.
   - **`Run the Gmail ops`** (Execute Sub-workflow → `LUMINA-TELEGRAM-GMAIL-OPS` **`LUCkYnz7GhjkyCtZ`**, Source Database, Mode « Run once with all items », **Wait For Sub-Workflow Completion = ON**).
   - **`Reply to user (gmail)`** (Telegram **Send Message**, credential **`LUMINA-Lyra - Telegram`**) : Chat ID `{{ $json.chat_id }}`, Text `{{ $json.text }}`, **Append n8n Attribution = OFF**, **Reply Markup / Parse Mode = None**.
3. Branche **FALSE** → **`Is it Maestro?`** (chaîne existante préservée).

Câblage effectué = 2 connexions ajoutées (web/ops FALSE → Gmail IF ; Gmail IF FALSE → Is it Maestro?) + 1 supprimée (l'ancienne web/ops FALSE → Is it Maestro?), **chacune vérifiée via le DOM** (arêtes Vue Flow). Gateway **republiée** (version de rollback créée).

## 5. Recette (live Telegram, validé 2026-07-13)

- **T1** « résume mes emails d'aujourd'hui » → résumé regroupé **B2C / B2B**, fidèle, pas d'erreur « Responses API ». ✅
- **T2** « des mails non lus côté partenaires ? » → `is:unread` appliqué (MVP interroge les 2 boîtes — filtrage par boîte nommée = post-MVP). ✅
- **T3** « prépare un brouillon à `frescatzi@gmail.com` … » → brouillon **réellement créé** dans Gmail → Brouillons de `hello@`, adresse extraite fidèlement (règle « ne jamais inventer d'adresse » respectée), mention « jamais envoyé ». ✅
- **T5** aucun envoi — **garanti par construction** (aucun node d'envoi dans le sous-wf). ✅
- **T6** non-régression : `memory_search` → branche mémoire ✅ ; `web/ops` → branche web (le « no response » = problème Hermès/latence connu, indépendant de M5) ✅ ; `maestro` → clarification + spécialistes ✅. Aucun débordement vers Gmail. ✅
- **T7** aucune erreur `can't parse entities`, aucun JSON brut. ✅

## 6. Difficultés rencontrées

1. **Session n8n perdue dans le navigateur piloté** : appels API en `Unauthorized` malgré une page affichée en cache ; il a fallu **relancer Chrome** puis se reconnecter dans l'onglet piloté (piège figé confirmé). Auth OK ensuite pour l'UI ; le `fetch` brut reste 401 (header `browser-id` manquant, non extractible car filtré).
2. **Auto-assignation de credential Gmail erronée** : à l'import, n8n a auto-assigné **PARTNERS** au node **Gmail B2C** (une seule credential auto-matchée quand il y en a plusieurs = aléatoire). Idem pour les Draft.
3. **`Use Responses API` absent** de la version du node `lmChatOpenAi` (options : Frequency/Max Tokens/Response Format/Presence/Temperature/Reasoning Effort/Timeout/Max Retries/Top P) — le guide supposait ce toggle.
4. **Modal Settings du workflow refusait de s'ouvrir** via l'automatisation (⋮ → Settings) → Error Workflow + tags non posés en auto.
5. **Tracé des connexions au pixel peu fiable** : les points de sortie (true/false) sont trop petits ; un drag « à l'estime » a échoué (25 px de décalage).
6. **Sélection d'arête ≠ intuitive** : cliquer une arête ne la sélectionne pas ; risque de **supprimer un node** si un node restait sélectionné (Backspace).

## 7. Solutions implémentées

1. **Reconnexion dans l'onglet piloté** (Karter fait le « Sign in » — l'authentification n'est pas automatisée), puis navigation en **clics internes SPA** uniquement.
2. **Construction par import JSON** : workflow complet écrit en `.json`, injecté via `navigator.clipboard.writeText` + **Cmd+V** (paste-import), puis **`file_upload`** sur l'input caché de n8n (« Import from file »). Une opération courte, résiliente aux coupures de session, vs 150 clics. Credentials **assignées ensuite** (dropdown : ouvrir → **taper pour filtrer** → cliquer le résultat au pixel exact).
3. **`gpt-5-mini` laissé en API standard** (Responses API indisponible) — validé fonctionnel au test T1. À reconsidérer seulement si un modèle Responses-only est requis.
4. **Error Workflow + tags** reportés en **réglage manuel** (2 min côté Karter).
5. **Coordonnées exactes des points de connexion via le DOM** (`getBoundingClientRect` des `.vue-flow__handle`, converties à l'échelle capture = **×0.94** ici : viewport DOM 1055×1318 vs capture 992×1239). Chaque connexion **vérifiée via le DOM** après tracé (parse des `data-id` d'arêtes `[src/outputs/main/N][tgt/inputs/main/M]`).
6. **Suppression d'arête via le bouton de survol** : `hover` sur le vrai **milieu de courbe** (`path.getPointAtLength(L/2)` + `getScreenCTM`), puis clic sur le `[data-test-id="delete-connection-button"]` (position lue au DOM). **Toujours vérifier la sélection au DOM avant tout Backspace.**
7. **Réutilisation par duplication** de la branche web/hermès (copier/coller de 4 nodes → credentials Telegram + expressions anti-clobbering conservés à l'identique) ; seuls changements : retarget Execute → sous-wf Gmail, ajout du champ `intent`, renommages.

## 8. Lessons learned

1. **n8n auto-assigne une credential quand il n'y en a qu'une compatible** — mais avec **plusieurs** (2 Gmail), l'assignation est aléatoire : **vérifier chaque node** (Gmail B2C = HELLO, Gmail B2B = PARTNERS, Draft idem).
2. **Deux credentials Telegram distinctes** dans ce stack : **Send Chat Action = `LuminaOsBot`**, **Send Message = `LUMINA-Lyra`**. Mirrorer exactement (dupliquer les nodes existants > réauthorer à l'aveugle).
3. **Piège figé `$json` clobbering** : après `Send typing`, `$json` = réponse API Telegram → tous les nodes suivants lisent le contrat via **`$('Classify the intent').first().json.<champ>`**.
4. **Set v3.4** : valeur en **mode Expression** ; l'éditeur **auto-ferme les accolades** → taper `{{ … ` **sans** les `}}` finales (sinon `}} }}` en double).
5. **Import JSON = méthode la plus résiliente** pour créer beaucoup de nodes d'un coup (paste-import ou file_upload sur l'input caché). Le code des nodes doit éviter les **backticks littéraux** (fences → ``{3}`) pour rester injectable en template-literal.
6. **Tracé/suppression de connexions = piloter par le DOM** (handles + `delete-connection-button`), jamais à l'estime ; **vérifier la sélection avant Backspace**.
7. **`alwaysOutputData` sur les lectures Gmail** = clé pour gérer proprement la boîte vide (T4) : un item vide traverse et le résumé dit « rien trouvé ».
8. **Web « no response » ≠ M5** : c'est la latence/Hermès déjà documentée ; Karter **veut** le web mais **rapide** (~20-40 s, cible < 42 s de poll). Ne pas empiler de passes LLM. *(cf. Passation Hermès No-Response-Latence 09-07.)*
9. Rappels figés réappliqués : sortie Telegram **Attribution OFF + Parse Mode None** ; **aucun node d'envoi** en M5 (Draft: Create only) ; Execute Sub-workflow **Wait = ON**.

## 9. Références

- GUIDE-BUILD M5 v1.2 ; SPEC-BUILD Telegram v0.3 ; GUIDE-BUILD M4 v1.0 ; Bible 360° v2.5 (→ addendum v2.6).
- Workflows : sous-wf `LUMINA-TELEGRAM-GMAIL-OPS` **`LUCkYnz7GhjkyCtZ`** ; Gateway `uGF3pSv6aTK79cqi` ; Router `KTLwQi7ZKHDTexMZ` ; Sentinel `xzaH0uWy0idKVphF`.
- Générique associé : `POS-GENERIQUE_Branche-gmail-ops-telegram.md`.
- Fichier d'import du sous-wf : `LUMINA-TELEGRAM-GMAIL-OPS.json` (racine projet).

---

*POS EXACT M5 v1.0 — 2026-07-13. Validé live (T1–T7). Reste : Error Workflow Sentinel + tags (manuel). Invariant : brouillon oui, envoi non (M8). n8n fait foi.*
