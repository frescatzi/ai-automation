---
type: raw
title: "GUIDE-BUILD_LUMINA-M5-Gmail-Ops_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m5-gmail-ops_2026-07-12.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# GUIDE-BUILD — LUMINA-TELEGRAM · M5 GMAIL OPS

**Version :** v1.2 (12-07 : credentials au code marque **AFTRSN** — ASP = After Sun People = AFTRSN. v1.1 : boîtes réelles confirmées B2C `hello@` + B2B `partners@`. v1.0 : rédaction initiale.)
**Date :** 2026-07-12
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra » (@luminaLyraBot)
**Étape :** M5 — brancher les intents `gmail_search` / `gmail_read` / `draft_email` sur un sous-workflow Gmail dédié : **lire / résumer** sur les **deux boîtes business After Sun People**, et **préparer des brouillons** — **jamais d'envoi** (l'envoi attend M8).
**Boussoles :** SPEC-BUILD Telegram v0.3 (M1/M2) · GUIDE-BUILD M4 v1.0 · Bible 360° v2.5. **n8n fait foi.**

**Décisions de cadrage (tranchées le 12-07) :**
1. **Workflow dédié** `LUMINA-TELEGRAM-GMAIL-OPS` (Lyra = passerelle du socle, découplée des agents de marque — on ne réutilise **pas** AFTRSN-Secretary).
2. **Deux boîtes business After Sun People** (`www.aftersunpeople.com`) :
   - **B2C** = `hello@aftersunpeople.com` (particuliers / clients)
   - **B2B** = `partners@aftersunpeople.com` (partenaires / pro)
3. **Lecture + brouillon uniquement.** Créer un brouillon Gmail n'expose rien vers l'extérieur → action **non critique**, autorisée en M5. **L'envoi = M8** (garde-fou de validation).

---

## 1. Vue d'ensemble

Deux chantiers, comme M4 :

- **Partie A** — créer le sous-workflow `LUMINA-TELEGRAM-GMAIL-OPS` : comprend la demande, interroge Gmail (les 2 boîtes), résume **ou** crée un brouillon, renvoie `{ chat_id, text }`.
- **Partie B** — insérer la branche M5 dans le dispatch de la Gateway (`uGF3pSv6aTK79cqi`), **sur la sortie FALSE de `Is it a web/ops question?`** (le maillon M4), en remplacement du stub pour les intents Gmail.

**Contrat d'entrée du sous-wf** (depuis la Gateway) : `{ intent, query, brand, chat_id, message_id }`.
**Contrat de sortie** : `{ chat_id, text }` (même forme que M3/M4 → le node Telegram de la Gateway ne change pas).

**Router : aucune modification nécessaire.** L'enum du classifieur (WF2 `KTLwQi7ZKHDTexMZ`) émet déjà `gmail_search`, `gmail_read`, `draft_email` (SPEC-BUILD Telegram §6.1). On se contente de les **brancher**. → *Vérifier en live que ces 3 valeurs sortent bien du Router avant de câbler (n8n fait foi).*

---

## 2. Prérequis terrain (à faire AVANT les nodes)

| # | Étape | Détail | DoD |
|---|---|---|---|
| P1 | Credential Gmail B2C | n8n → Credentials → **Gmail OAuth2** pour `hello@aftersunpeople.com`. Scopes : lecture + **création de brouillon** (`gmail.readonly` + `gmail.compose`, ou `gmail.modify`). Nommer **`Gmail - AFTRSN-B2C`**. | Credential créé + « Connection tested » vert |
| P2 | Credential Gmail B2B | Idem pour `partners@aftersunpeople.com`. Nommer **`Gmail - AFTRSN-B2B`**. | Credential créé + testé |
| P3 | Confirmer les intents | Depuis Telegram : « résume mes emails d'aujourd'hui » et « prépare un brouillon à X » → vérifier via une exécution du Router que `intent ∈ {gmail_search, gmail_read, draft_email}`. | 3 intents observés |
| P4 | Sentinel | Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH` (`xzaH0uWy0idKVphF`) à régler sur le nouveau sous-wf (Settings → Error Workflow). | Réglé |

> **Sécurité (invariant Bible J.2)** : n'activer **aucune** opération d'envoi Gmail. On n'utilise que **Message: Get Many / Get** (lecture) et **Draft: Create** (brouillon). Le node « Send » est proscrit en M5.

---

## 3. Partie A — Sous-workflow `LUMINA-TELEGRAM-GMAIL-OPS`

Nouveau workflow, projet **Personal**. Tags : `TEST→PROD · BRAIN · SHARED · ASSISTANT-BOT`. **Settings → Error Workflow = `xzaH0uWy0idKVphF`.**
**ID réel à consigner ici après création : `__________`.**

> Piège figé (M4) : ce sous-wf a un **Execute Workflow Trigger** → il n'apparaîtra **jamais** dans le MCP n8n (le MCP n'expose que les workflows publiés à trigger Schedule/Webhook/Form/Chat). Vérification par canvas / JSON, pas par MCP.

**Flux :**
```
Receive from Gateway → Understand request (LLM) → Parse plan → Route op (Switch)
   ├─ read  → [Gmail B2C + Gmail B2B] → Merge → Summarize (LLM) → Shape reply (read)
   └─ draft → Compose draft (LLM) → Route mailbox (Switch) → Gmail Create Draft → Shape reply (draft)
```

### N1 — `Receive from Gateway`
- **Execute Workflow Trigger** (`executeWorkflowTrigger`), **Input Source = Accept all data**. Reçoit `{ intent, query, brand, chat_id, message_id }`.

### N2 — `Understand request` (LLM)
- **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`), **Prompt = Define below**.
- **Sous-node modèle** : **OpenAI Chat Model** (`lmChatOpenAi`), credential **`OpenAi account`**, **Model = `gpt-5-mini`** (From list), **Use Responses API = ON**, température basse.
- **Prompt** (renvoie **UNIQUEMENT** un JSON) :
```
Tu es le planificateur Gmail de Lyra (assistante privée de Karter, LUMINA OS / AFTER SUN PEOPLE).
Deux boites business existent :
- "b2c" = hello@aftersunpeople.com (particuliers / clients)
- "b2b" = partners@aftersunpeople.com (partenaires / pro)
A partir de la demande, renvoie UNIQUEMENT un JSON valide, sans texte autour :
{
  "op": "read|draft",
  "mailbox": "b2c|b2b|both",
  "gmail_query": "<operateurs de recherche Gmail, ex: newer_than:2d is:unread from:...>",
  "max": 10,
  "draft": { "to": "", "subject": "", "body_points": "" }
}
Regles :
- op="draft" seulement si la demande est de PREPARER/ECRIRE un mail ; sinon op="read".
- mailbox : "both" par defaut en lecture si aucune boite n'est nommee ; "b2b" si contexte partenaires/pro, "b2c" si clients/particuliers. Pour un brouillon, choisir la boite pertinente (defaut "b2c").
- gmail_query : traduis la demande en operateurs Gmail (from:, subject:, newer_than:, is:unread, has:attachment...). Si rien de precis : "newer_than:2d".
- max : 1 a 20 (defaut 10).
- draft.to / subject / body_points : remplis-les seulement si op="draft" (body_points = notes/points cles, PAS le mail final).
- Ne jamais inventer d'adresse : si le destinataire est inconnu, laisse "to" vide.
Reponds en francais uniquement dans les textes ; le JSON reste en anglais pour les cles.
```
- **User prompt** : `intent={{ $json.intent }} | message={{ $json.query }}`.

### N3 — `Parse plan` (Code, Run Once for All Items)
```javascript
/**
 * N3 — Parse plan Gmail
 * Entrée : sortie LLM (texte JSON) + contexte du trigger.
 * Sortie : plan normalisé + contexte repropagé.
 */
const ctx = $('Receive from Gateway').first().json;
let p;
try { p = JSON.parse(($json.text ?? $json.response ?? '{}').replace(/^```json|```$/g, '').trim()); }
catch (e) { p = {}; }

const op = (p.op === 'draft') ? 'draft' : 'read';
let mailbox = ['b2c', 'b2b', 'both'].includes(p.mailbox) ? p.mailbox : 'both';
if (op === 'draft' && mailbox === 'both') mailbox = 'b2c'; // un brouillon = une seule boîte
let max = parseInt(p.max, 10); if (!Number.isFinite(max)) max = 10; max = Math.min(Math.max(max, 1), 20);
const gmail_query = (p.gmail_query && String(p.gmail_query).trim()) || 'newer_than:2d';
const draft = p.draft ?? { to: '', subject: '', body_points: '' };

return [{ json: {
  op, mailbox, max, gmail_query,
  draft,
  user_request: ctx.query ?? '',
  chat_id: ctx.chat_id, message_id: ctx.message_id
} }];
```

### N4 — `Route op` (Switch sur `{{ $json.op }}`)
- Sortie **`read`** → branche lecture (N5a/N5b).
- Sortie **`draft`** → branche brouillon (N9).

---

### Branche LECTURE (résumé des 2 boîtes)

> MVP : on interroge **toujours les deux boîtes** en parallèle, on **tague** l'origine, on fusionne, puis on résume en tenant compte de la demande d'origine. (Filtrage par boîte nommée = raffinement post-MVP, cf. §6.)

### N5a — `Gmail B2C` (Gmail, Message: Get Many)
- Credential **`Gmail - AFTRSN-B2C`**. Resource **Message**, Operation **Get Many**.
- **Return All = OFF**, **Limit** = `{{ $('Parse plan').first().json.max }}`.
- **Filters → Search (`q`)** = `{{ $('Parse plan').first().json.gmail_query }}`.
- **Options** : **Simplify = ON** (renvoie from/subject/snippet/date — suffisant pour résumer, évite le N+1 fetch des corps).

### N5a' — `Tag b2c` (Edit Fields/Set v3.4, **valeurs en mode Expression**, Include Other Input Fields = ON)
- `account` (Fixed autorisé ici, texte court) = `b2c`.

### N5b — `Gmail B2B` (Gmail, Message: Get Many)
- Credential **`Gmail - AFTRSN-B2B`**. Mêmes réglages que N5a (Limit, `q`, Simplify ON).

### N5b' — `Tag b2b` (Set) — `account` = `b2b`.

### N6 — `Merge` (mode **Append**) ← entrées N5a' et N5b'.

### N7 — `Summarize` (LLM, Basic LLM Chain, `gpt-5-mini`, cred `OpenAi account`, Responses API ON)
- **Prompt = Define below**, **grounding strict** :
```
Tu es Lyra, l'assistante personnelle de Karter. Voici des emails de deux boites business
(champ "account" : b2c = hello@aftersunpeople.com / b2b = partners@aftersunpeople.com).
Fais un resume clair pour Telegram, en francais.
DEMANDE DE KARTER : {{ $('Parse plan').first().json.user_request }}

REGLES STRICTES : n'invente rien ; n'utilise que les emails fournis ; conserve exp., objets et dates tels quels.
Regroupe par boite (B2C / B2B). Pour chaque email pertinent : expediteur, objet, 1 ligne de contexte.
Si aucune boite n'a d'email pertinent, dis-le simplement. Ne mets pas de balises HTML ; utilise des crochets [..] si besoin, pas de <..>.

EMAILS (JSON) :
{{ JSON.stringify($input.all().map(i => i.json)) }}
```
> Piège figé : après un node LangChain, lire le contexte via **`.first()`** (pas `.item`). Ici on lit `Parse plan` via `.first()` et les emails via `$input.all()`.

### N8 — `Shape reply (read)` (Code, Run Once for All Items)
```javascript
const ctx = $('Parse plan').first().json;
let text = String($json.text ?? $json.response ?? '').trim();
if (!text) text = "Je n'ai rien trouvé à résumer pour cette demande.";
text = text.replace(/\n{3,}/g, '\n\n');
if (text.length > 3800) text = text.slice(0, 3800) + '…'; // marge sous la limite Telegram (4096)
return [{ json: { chat_id: ctx.chat_id, text } }];
```

---

### Branche BROUILLON (jamais d'envoi)

### N9 — `Compose draft` (LLM, Basic LLM Chain, `gpt-5-mini`, cred `OpenAi account`, Responses API ON)
- **Prompt = Define below** :
```
Tu es Lyra, l'assistante personnelle de Karter (LUMINA OS / AFTER SUN PEOPLE).
Redige un BROUILLON d'email en francais, ton professionnel et chaleureux, a partir des points ci-dessous.
Renvoie UNIQUEMENT un JSON : { "subject": "...", "body": "..." }.
Ne mets pas de signature d'envoi automatique ; ce n'est qu'un brouillon a relire.

DESTINATAIRE : {{ $('Parse plan').first().json.draft.to }}
OBJET SUGGERE : {{ $('Parse plan').first().json.draft.subject }}
POINTS CLES : {{ $('Parse plan').first().json.draft.body_points }}
```
> Note évolutive : en **M7 (Draft Writer, voix de marque)**, ce node sera remplacé/complété par un appel à `LUMINA-TELEGRAM-DRAFT-WRITER` pour appliquer le ton exact de la marque (B2C chaleureux vs B2B pro). En M5, voix « Lyra » suffit (brouillon relu avant tout envoi).

### N10 — `Parse draft` (Code) — parse le JSON `{subject, body}`, repropage `mailbox`, `draft.to`, `chat_id`.
```javascript
const ctx = $('Parse plan').first().json;
let d; try { d = JSON.parse(($json.text ?? '{}').replace(/^```json|```$/g,'').trim()); } catch(e){ d = {}; }
return [{ json: {
  to: ctx.draft?.to ?? '',
  subject: (d.subject ?? ctx.draft?.subject ?? '(sans objet)'),
  body: (d.body ?? ''),
  mailbox: ctx.mailbox,
  chat_id: ctx.chat_id
} }];
```

### N11 — `Route mailbox` (Switch sur `{{ $json.mailbox }}`) → `b2c` / `b2b`.

### N12a — `Create Draft B2C` (Gmail, Resource **Draft**, Operation **Create**)
- Credential **`Gmail - AFTRSN-B2C`**.
- **Subject** = `{{ $json.subject }}` · **Message** = `{{ $json.body }}` · **To** = `{{ $json.to }}` (Options → Send To).
- ⚠️ **Operation = Create (Draft)** — **jamais** Send.

### N12b — `Create Draft B2B` — idem avec credential **`Gmail - AFTRSN-B2B`**.

### N13 — `Shape reply (draft)` (Code, Run Once for All Items)
```javascript
const src = $('Parse draft').first().json;
const box = src.mailbox === 'b2b' ? 'B2B (partners@)' : 'B2C (hello@)';
const to  = src.to ? ` a ${src.to}` : '';
const text = `[Brouillon cree dans la boite ${box}${to}] (jamais envoye)\n\n`
           + `Objet : ${src.subject}\n\n`
           + `${src.body}\n\n`
           + `A relire / valider avant tout envoi (l'envoi passera par la validation M8).`;
return [{ json: { chat_id: src.chat_id, text: text.slice(0, 3800) } }];
```

**Sortie du sous-workflow** (branche exécutée) = **`{ chat_id, text }`**.

---

## 4. Partie B — Câblage dans le dispatch de la Gateway

Rappel structure actuelle (post-M4) : `IF clarification` → FALSE → `Question for Memory-Search ?` → FALSE → **`Is it a web/ops question?`** → TRUE = branche web/hermes / **FALSE = `Reply (module pending)`** (stub).

On insère M5 **sur la sortie FALSE de `Is it a web/ops question?`** (avant le stub).

> ⚠️ **Piège `$json` clobbering** (M4) : le node `Send typing` **remplace `$json`** par la réponse de l'API Telegram. Tous les nodes après `Send typing` lisent le contrat via **`$('Classify the intent').first().json.<champ>`** (confirmer le nom exact du node qui appelle le Router dans la Gateway live), **pas** `$json`.

### B1 — `Is it a Gmail question?` (IF, conditions **String**, combinateur **OR**)
- `{{ $json.intent }}` **is equal to** `gmail_search`
- `{{ $json.intent }}` **is equal to** `gmail_read`
- `{{ $json.intent }}` **is equal to** `draft_email`
- Entrée branchée sur la sortie **FALSE** de `Is it a web/ops question?`.

> **Décision interim (D1)** : `draft_email` est rattaché à M5 (branche brouillon) tant que **M7** n'est pas livré. Créer un brouillon est sans risque (aucun envoi). M7 viendra ensuite raffiner la **voix** de la rédaction. Si tu préfères réserver `draft_email` à M7, retire cette 3ᵉ condition et garde seulement `gmail_search`/`gmail_read`.

### B2 (TRUE) — `Send typing` (Telegram, Send Chat Action)
- Credential `LUMINA-Lyra - Telegram`. **Chat ID** = `{{ $('Classify the intent').first().json.chat_id }}`, **Action** = `typing`.

### B3 — `Prepare Input for Gmail` (Edit Fields/Set, **Keep Only Set Fields**, mode **Expression**, repropagé depuis le Router)
- `intent` = `{{ $('Classify the intent').first().json.intent }}`
- `query` = `{{ $('Classify the intent').first().json.query }}`
- `brand` = `{{ $('Classify the intent').first().json.brand }}`
- `chat_id` = `{{ $('Classify the intent').first().json.chat_id }}`
- `message_id` = `{{ $('Classify the intent').first().json.message_id }}`

### B4 — `Run the Gmail ops` (Execute Sub-workflow → `LUMINA-TELEGRAM-GMAIL-OPS`, **Wait for completion = ON**). Sortie = `{ chat_id, text }`.

### B5 — `Reply to user (gmail)` (Telegram, Send Message)
- Credential `LUMINA-Lyra - Telegram`. **Chat ID** = `{{ $json.chat_id }}`, **Text** = `{{ $json.text }}`.
- **Append n8n Attribution = OFF** · **Parse Mode = None**.

### B6 (FALSE) — stub inchangé : la sortie FALSE de `Is it a Gmail question?` reste sur `Reply (module pending)` (les intents `calendar_*` etc. attendent M6+).

---

## 5. Recette (test live depuis Telegram, bot Lyra)

| # | Envoi | Attendu |
|---|---|---|
| T1 | « résume mes emails d'aujourd'hui » | « Lyra écrit… » puis résumé **regroupé par boîte** (B2C / B2B), fidèle (aucun email inventé). |
| T2 | « des mails non lus côté partenaires ? » | `mailbox=b2b`, `gmail_query` avec `is:unread`, résumé pertinent de partners@. |
| T3 | « prépare un brouillon à [X] pour [objet] » | Brouillon **créé dans Gmail** (boîte annoncée), texte renvoyé dans Telegram, mention « jamais envoyé ». **Vérifier dans Gmail → Brouillons.** |
| T4 | Boîte vide / rien de pertinent | Message propre « rien trouvé », pas d'erreur brute. |
| T5 | **Aucun envoi** ne part jamais | Contrôler qu'aucun mail n'est envoyé (seulement des brouillons). |
| T6 | `memory_search`, `web_search`, clarification | **inchangés** (non-régression M3/M4). |
| T7 | Sortie propre | Aucune erreur `can't parse entities` (Parse Mode None), aucun JSON brut affiché. |

**DoD M5 :** T1–T5 passent ; T6 non-régression OK ; brouillons visibles dans les 2 boîtes ; zéro envoi.

---

## 6. Points à trancher au build

1. ~~Adresse exacte boîte #2~~ **Tranché (12-07)** : B2C `hello@aftersunpeople.com` + B2B `partners@aftersunpeople.com` (`www.aftersunpeople.com`).
2. **Scopes OAuth** — recommandé : `gmail.readonly` + `gmail.compose` (lecture + brouillon, **pas d'envoi**). `gmail.modify` possible mais plus large.
3. **Filtrage par boîte nommée en lecture** — MVP interroge toujours les 2 boîtes. Si tu veux « seulement B2B » quand tu le nommes → ajouter 2 IF (skip la boîte non demandée) en post-MVP (le champ `mailbox` du plan est déjà prêt pour ça).
4. **`draft_email` → M5 ou M7 ?** — D1 le rattache à M5 (interim). À rebasculer vers M7 quand le Draft Writer voix-de-marque sera prêt.
5. **Modèle LLM** — `gpt-5-mini` retenu (cohérent M3/M4, rapide/bon marché). À confirmer.

---

## 7. Pièges à respecter (figés — ne pas les réapprendre)

1. **Aucune opération d'envoi Gmail** en M5. Uniquement Message Get/Get Many + Draft Create. L'envoi = M8.
2. **Set v3.4** : toute valeur `{{ }}` en **mode Expression** (Fixed → texte littéral → bug « chat not found »).
3. **Après un node LangChain** : lire via **`.first()`** (pas `.item`).
4. **Sortie Telegram** : **Attribution OFF** + **Parse Mode None** ; crochets `[..]` pas `<..>` (avalés par le HTML Telegram).
5. **`$json` clobbering** après `Send typing` → relire le contrat via `$('Classify the intent').first().json`.
6. **Execute Sub-workflow** : **Wait for completion = ON** (sinon pas de `{chat_id,text}` en retour).
7. **Execute Workflow Trigger** ⇒ sous-wf **invisible en MCP** : vérifier par canvas/JSON.
8. **Gmail Simplify = ON** en lecture (snippets suffisent → pas de N+1 fetch des corps ; plus rapide).
9. **Ne pas empiler** de passes LLM au-delà du nécessaire (piège latence M3) : 1 passe comprendre + 1 passe résumer/rédiger, pas plus.

---

## 8. Checklist de build (ordre d'exécution)

1. [ ] P1–P2 : credentials `Gmail - AFTRSN-B2C` (hello@) + `Gmail - AFTRSN-B2B` (partners@) créées et testées.
2. [ ] P3 : confirmer en live que le Router émet `gmail_search`/`gmail_read`/`draft_email`.
3. [ ] Créer `LUMINA-TELEGRAM-GMAIL-OPS` (Personal, Error Workflow = Sentinel) — **consigner l'ID réel**.
4. [ ] N1 Execute Workflow Trigger (Accept all data).
5. [ ] N2 `Understand request` (gpt-5-mini → plan JSON) + N3 `Parse plan`.
6. [ ] N4 `Route op` (Switch read/draft).
7. [ ] Lecture : N5a/N5b Gmail Get Many (B2C + B2B, `q`, Simplify) + tags + N6 Merge + N7 Summarize + N8 Shape reply.
8. [ ] Brouillon : N9 Compose + N10 Parse + N11 Route mailbox + N12a/b Create Draft + N13 Shape reply.
9. [ ] Gateway Partie B : `Is it a Gmail question?` + `Send typing` + `Prepare Input for Gmail` + `Run the Gmail ops` + `Reply to user (gmail)`, sur la sortie FALSE de `Is it a web/ops question?`.
10. [ ] Tests §5 en live depuis Telegram (dont : brouillons visibles dans les 2 boîtes, zéro envoi, non-régression M3/M4).
11. [ ] **À validation** → convention documentaire (§9).

---

## 9. Après validation (convention §3 de la passation — automatique)

Produire et **double-sauvegarder** (projet Claude `Marches-a-suivre/` **+** Drive `LUMINA AI DOCS`) :
- `POS-LUMINA_Telegram-Gmail-Ops_2026-07-12.md` (**exact** : IDs réels, noms de nodes, expressions, credentials) — sections **Difficultés / Solutions / Lessons learned**.
- `POS-GENERIQUE_Branche-gmail-ops-telegram.md` (**patron réutilisable**) — mêmes 3 sections.

Puis : **addendum Bible v2.6** (nouveau sous-wf M5 + IDs + invariant « brouillon oui, envoi non »), **cocher M5** dans le SPEC-BUILD / la passation, **mettre à jour la mémoire** (`lumina-personal-command-bot`).

---

*GUIDE-BUILD M5 v1.1 — 2026-07-12. Karter construit dans n8n et valide. Contrats Gateway/Router repris de M4 v1.0 + SPEC-BUILD Telegram v0.3 ; à re-vérifier en live (n8n fait foi). Boussoles : SPEC-BUILD Telegram v0.3 · GUIDE-BUILD M4 v1.0 · Bible 360° v2.5.*
