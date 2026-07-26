---
type: raw
title: "GUIDE-BUILD_LUMINA-M6-Calendar-Ops_2026-07-13"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m6-calendar-ops_2026-07-13.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# GUIDE-BUILD — LUMINA-TELEGRAM · M6 CALENDAR OPS

**Version :** v1.0 (13-07 : rédaction initiale, après vérif live des credentials).
**Date :** 2026-07-13
**Projet :** LUMINA OS · Personal Assistant Bot — bot « Lyra » (@luminaLyraBot)
**Étape :** M6 — brancher l'intent `calendar_read` sur un sous-workflow Calendar dédié : **consulter / résumer** les **deux agendas business After Sun People** (agenda du jour, prochains RDV) — **lecture seule**. La **création/modification d'événement** (`calendar_action`, surtout avec invités) est **hors MVP** et **gatée M8**.
**Boussoles :** SPEC-BUILD Telegram v0.3 (M1/M2) · GUIDE-BUILD M5 v1.2 · Bible 360° v2.5 + addendum v2.6. **n8n fait foi.**

**Décisions de cadrage (tranchées le 13-07, après vérif live) :**
1. **Workflow dédié** `LUMINA-TELEGRAM-CALENDAR-OPS` (Lyra = passerelle du socle, découplée des agents de marque — on ne réutilise **pas** AFTRSN-Secretary, cohérent avec M5).
2. **Deux agendas business** (symétrie M5, credentials **déjà en place et vérifiées le 13-07**) :
   - **B2C** = agenda de `hello@aftersunpeople.com` → credential **`Hello AFTRSN Calendar`** (Google Calendar OAuth2 API)
   - **B2B** = agenda de `partners@aftersunpeople.com` → credential **`PARTNERS-AFTRSN Calendar account`** (Google Calendar OAuth2 API)
   - *(L'agenda **personnel** `itiskarter@gmail` n'a pas encore de credential n8n → hors MVP ; il rejoindra le « hub perso » du backlog quand il sera cadré.)*
3. **Lecture seule.** Uniquement **Event : Get Many** (consultation). **Aucun** Create/Update/Delete d'événement en M6. La création d'événement (surtout avec invités = envoi d'invitation, action critique) attend **M8** (garde-fou de validation).

---

## 1. Vue d'ensemble

Deux chantiers, comme M5 :

- **Partie A** — créer le sous-workflow `LUMINA-TELEGRAM-CALENDAR-OPS` : comprend la demande (période, agenda), interroge les 2 agendas, résume, renvoie `{ chat_id, text }`.
- **Partie B** — insérer la branche M6 dans le dispatch de la Gateway (`uGF3pSv6aTK79cqi`), **sur la sortie FALSE de `Is it a Gmail question?`** (le maillon M5), avant `Is it Maestro?`.

**Contrat d'entrée du sous-wf** (depuis la Gateway) : `{ intent, query, brand, chat_id, message_id }`.
**Contrat de sortie** : `{ chat_id, text }` (même forme que M3/M4/M5 → le node Telegram de la Gateway ne change pas).

**Router : aucune modification nécessaire.** L'enum du classifieur (WF2 `KTLwQi7ZKHDTexMZ`) émet déjà `calendar_read` et `calendar_action` (SPEC-BUILD Telegram §6.1). On se contente de les **brancher**. → *Vérifier en live que `calendar_read` sort bien du Router avant de câbler (n8n fait foi) — cf. P3.*

---

## 2. Prérequis terrain (à faire AVANT les nodes)

| # | Étape | Détail | DoD |
|---|---|---|---|
| P1 | Credential Calendar B2C | **Déjà en place (vérifié 13-07)** : `Hello AFTRSN Calendar` (Google Calendar OAuth2 API, hello@). Vérifier « Connection tested » vert. | Credential OK |
| P2 | Credential Calendar B2B | **Déjà en place (vérifié 13-07)** : `PARTNERS-AFTRSN Calendar account` (Google Calendar OAuth2 API, partners@). | Credential OK |
| P3 | Confirmer l'intent | Depuis Telegram : « mon agenda aujourd'hui » / « mes prochains rendez-vous » → vérifier via une exécution du Router que `intent = calendar_read`. | intent observé |
| P4 | Timezone | **Tranché (13-07) : `Europe/Paris`.** La Bible note « Default Timezone not valid » dans les Settings n8n → on **calcule les bornes en JS avec luxon (`Europe/Paris`)**, DST géré automatiquement, on ne se fie pas au défaut n8n. | Fuseau fixé ✅ |
| P5 | Sentinel | Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH` (`xzaH0uWy0idKVphF`) à régler sur le nouveau sous-wf (Settings → Error Workflow). | Réglé |

> **Sécurité (invariant Bible)** : n'activer **aucune** opération d'écriture Calendar. On n'utilise que **Event : Get Many** (lecture). Les opérations Create/Update/Delete d'événement sont proscrites en M6 (→ M8).

---

## 3. Partie A — Sous-workflow `LUMINA-TELEGRAM-CALENDAR-OPS`

Nouveau workflow, projet **Personal**. Tags : `PROD · BRAIN · SHARED · ASSISTANT-BOT` (statut PROD quand validé et en service ; poser `TEST` d'abord si tu préfères recetter en TEST). **Settings → Error Workflow = `xzaH0uWy0idKVphF`.**
**ID réel à consigner ici après création : `__________`.**

> Piège figé (M4/M5) : ce sous-wf a un **Execute Workflow Trigger** → il n'apparaîtra **jamais** dans le MCP n8n (le MCP n'expose que les workflows à trigger Schedule/Webhook/Form/Chat). Vérification par canvas / JSON, pas par MCP.

**Flux :**
```
Receive from Gateway → Understand request (LLM) → Parse plan → Route op (Switch)
   ├─ read   → [Cal B2C + Cal B2B (Get Many events)] → tags → Merge → Summarize (LLM) → Shape reply (read)
   └─ action → Shape reply (gated M8)   ← message poli « création pas encore dispo », AUCUNE écriture
```

### N1 — `Receive from Gateway`
- **Execute Workflow Trigger** (`executeWorkflowTrigger`), **Input Source = Accept all data**. Reçoit `{ intent, query, brand, chat_id, message_id }`.

### N2 — `Understand request` (LLM)
- **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`), **Prompt = Define below**.
- **Sous-node modèle** : **OpenAI Chat Model** (`lmChatOpenAi`), credential **`OpenAi account`**, **Model = `gpt-5-mini`** (From list). *(Rappel M5 : pas de toggle « Use Responses API » dans cette version → API standard, valide.)*
- **Prompt** (renvoie **UNIQUEMENT** un JSON, période **symbolique** — le calcul des dates se fait en JS au N3, pas par le LLM) :
```
Tu es le planificateur Agenda de Lyra (assistante privee de Karter, LUMINA OS / AFTER SUN PEOPLE).
Deux agendas business existent :
- "b2c" = agenda de hello@aftersunpeople.com (particuliers / clients)
- "b2b" = agenda de partners@aftersunpeople.com (partenaires / pro)
A partir de la demande, renvoie UNIQUEMENT un JSON valide, sans texte autour :
{
  "op": "read|action",
  "calendar": "b2c|b2b|both",
  "range": "today|tomorrow|week|next|custom",
  "days_ahead": 7,
  "max": 10
}
Regles :
- op="read" par defaut. op="action" UNIQUEMENT si la demande est de CREER / MODIFIER / SUPPRIMER un evenement.
- calendar : "both" par defaut ; "b2b" si contexte partenaires/pro, "b2c" si clients/particuliers.
- range : "today" (aujourd'hui), "tomorrow" (demain), "week" (7 prochains jours), "next" (prochains RDV a venir), "custom" sinon.
- days_ahead : nombre de jours a couvrir pour range="next"/"week" (defaut 7, max 31).
- max : 1 a 20 (defaut 10).
- Ne jamais inventer. Reponds en francais dans les textes ; le JSON reste en anglais pour les cles.
```
- **User prompt** : `intent={{ $json.intent }} | message={{ $json.query }}`.

### N3 — `Parse plan` (Code, Run Once for All Items)
Normalise le plan **et calcule `time_min` / `time_max` en ISO 8601, fuseau `Europe/Paris`** via **luxon** (`DateTime` est exposé globalement dans les Code nodes n8n ; DST géré automatiquement).
```javascript
/**
 * N3 — Parse plan Calendar
 * Entrée : sortie LLM (texte JSON) + contexte du trigger.
 * Sortie : plan normalisé + bornes ISO (Europe/Paris, DST géré via luxon).
 */
const ctx = $('Receive from Gateway').first().json;
let p;
try { p = JSON.parse(($json.text ?? $json.response ?? '{}').replace(/^```json|```$/g, '').trim()); }
catch (e) { p = {}; }

const op = (p.op === 'action') ? 'action' : 'read';
let calendar = ['b2c', 'b2b', 'both'].includes(p.calendar) ? p.calendar : 'both';
let max = parseInt(p.max, 10); if (!Number.isFinite(max)) max = 10; max = Math.min(Math.max(max, 1), 20);
let days = parseInt(p.days_ahead, 10); if (!Number.isFinite(days)) days = 7; days = Math.min(Math.max(days, 1), 31);
const range = ['today','tomorrow','week','next','custom'].includes(p.range) ? p.range : 'next';

// Fuseau business = Europe/Paris (confirmé 13-07). luxon gère l'heure d'été/hiver.
const ZONE = 'Europe/Paris';
const now = DateTime.now().setZone(ZONE);
let timeMin = now, timeMax = now.plus({ days });
if (range === 'today')    { timeMin = now.startOf('day');   timeMax = now.endOf('day'); }
if (range === 'tomorrow') { const t = now.plus({ days: 1 }); timeMin = t.startOf('day'); timeMax = t.endOf('day'); }
if (range === 'week')     { timeMin = now;                   timeMax = now.plus({ days: 7 }); }
if (range === 'next')     { timeMin = now;                   timeMax = now.plus({ days }); }

return [{ json: {
  op, calendar, max, range,
  time_min: timeMin.toISO(), time_max: timeMax.toISO(),
  tz: ZONE,
  user_request: ctx.query ?? '',
  chat_id: ctx.chat_id, message_id: ctx.message_id
} }];
```
> Note : `DateTime` (luxon) est fourni globalement par n8n dans les Code nodes ; `.toISO()` renvoie une ISO avec offset que Google Calendar accepte pour Time Min/Max. Si jamais `DateTime` n'est pas dispo dans ta version, `require('luxon').DateTime` en secours.

### N4 — `Route op` (Switch sur `{{ $json.op }}`)
- Sortie **`read`** → branche lecture (N5a/N5b).
- Sortie **`action`** → branche gated (N9).

---

### Branche LECTURE (résumé des 2 agendas)

> MVP : on interroge **toujours les deux agendas** en parallèle, on **tague** l'origine, on fusionne, puis on résume en tenant compte de la demande. (Filtrage par agenda nommé = raffinement post-MVP ; le champ `calendar` du plan est déjà prêt pour ça.)

### N5a — `Cal B2C` (Google Calendar, Event : Get Many)
- Credential **`Hello AFTRSN Calendar`**. Resource **Event**, Operation **Get Many**.
- **Calendar** = agenda primaire du compte hello@ (sélectionner dans la liste ; souvent `hello@aftersunpeople.com`).
- **Return All = OFF**, **Limit** = `{{ $('Parse plan').first().json.max }}`.
- **Options** : **After (Time Min)** = `{{ $('Parse plan').first().json.time_min }}` · **Before (Time Max)** = `{{ $('Parse plan').first().json.time_max }}` · **Single Events = ON** (développe les récurrences en occurrences datées) · **Order By = startTime**.
- **`alwaysOutputData = ON`** (gère l'agenda vide sans casser le Merge).

### N5a' — `Tag b2c` (Edit Fields/Set v3.4, **valeurs en mode Expression**, Include Other Input Fields = ON)
- `account` (texte court, Fixed autorisé ici) = `b2c`.

### N5b — `Cal B2B` (Google Calendar, Event : Get Many)
- Credential **`PARTNERS-AFTRSN Calendar account`**. Mêmes réglages que N5a (Calendar = agenda partners@, Limit, Time Min/Max, Single Events, Order By, alwaysOutputData ON).

### N5b' — `Tag b2b` (Set) — `account` = `b2b`.

### N6 — `Merge` (mode **Append**) ← entrées N5a' et N5b'.

### N7 — `Summarize` (LLM, Basic LLM Chain, `gpt-5-mini`, cred `OpenAi account`)
- **Prompt = Define below**, **grounding strict** :
```
Tu es Lyra, l'assistante personnelle de Karter. Voici des evenements de deux agendas business
(champ "account" : b2c = hello@aftersunpeople.com / b2b = partners@aftersunpeople.com).
Fais un resume clair pour Telegram, en francais.
DEMANDE DE KARTER : {{ $('Parse plan').first().json.user_request }}

REGLES STRICTES : n'invente rien ; n'utilise que les evenements fournis ; conserve titres, dates et heures tels quels.
Affiche les heures en clair (heure de Paris, Europe/Paris). Regroupe par agenda (B2C / B2B), tri chronologique.
Pour chaque evenement : jour + heure de debut, titre, lieu si present, participants si presents.
Si un agenda n'a rien sur la periode, dis-le simplement. Pas de balises HTML ; crochets [..] si besoin, pas de <..>.

EVENEMENTS (JSON) :
{{ JSON.stringify($input.all().map(i => i.json)) }}
```
> Piège figé : après un node LangChain, lire le contexte via **`.first()`** (pas `.item`). Ici on lit `Parse plan` via `.first()` et les événements via `$input.all()`.

### N8 — `Shape reply (read)` (Code, Run Once for All Items)
```javascript
const ctx = $('Parse plan').first().json;
let text = String($json.text ?? $json.response ?? '').trim();
if (!text) text = "Rien de prévu sur cette période dans les deux agendas.";
text = text.replace(/\n{3,}/g, '\n\n');
if (text.length > 3800) text = text.slice(0, 3800) + '…'; // marge sous la limite Telegram (4096)
return [{ json: { chat_id: ctx.chat_id, text } }];
```

---

### Branche ACTION (gated M8 — aucune écriture au MVP)

### N9 — `Shape reply (gated)` (Code, Run Once for All Items)
Répond poliment sans rien écrire dans l'agenda. Cette branche sera recâblée vers la **validation M8** quand elle existera.
```javascript
const ctx = $('Parse plan').first().json;
const text = "Je peux consulter tes agendas (aujourd'hui, prochains RDV…), "
  + "mais la création ou la modification d'événement n'est pas encore disponible : "
  + "elle passera par la validation (M8). Dis-moi ce que tu veux, je te le prépare quand ce sera prêt.";
return [{ json: { chat_id: ctx.chat_id, text } }];
```

**Sortie du sous-workflow** (branche exécutée) = **`{ chat_id, text }`**.

---

## 4. Partie B — Câblage dans le dispatch de la Gateway

Rappel structure actuelle (post-M5) : `IF clarification` → FALSE → `Question for Memory-Search ?` → FALSE → `Is it a web/ops question?` → FALSE → **`Is it a Gmail question?`** → TRUE = branche Gmail / **FALSE = `Is it Maestro?`** → FALSE = `Reply (module pending)` (stub).

On insère M6 **sur la sortie FALSE de `Is it a Gmail question?`**, **avant `Is it Maestro?`**.

> **Câblage réel confirmé en live (13-07)** : aujourd'hui `Is it a Gmail question?` → **sortie 0 (TRUE)** = `Send typing (gmail)` ; **sortie 1 (FALSE)** = `Is it Maestro?`. Après insertion : sortie 1 (FALSE) de `Is it a Gmail question?` → **`Is it a Calendar question?`** ; TRUE de celui-ci → branche calendar ; **FALSE → `Is it Maestro?`** (rebrancher l'ancien lien). Gateway = `uGF3pSv6aTK79cqi` (34 nodes avant M6). **Astuce fiable** : sélectionner la branche Gmail (les 5 nodes) puis Ctrl/Cmd+C / Ctrl/Cmd+V pour dupliquer en héritant des 2 credentials Telegram + expressions, puis ne changer que l'IF (conditions calendar_read/calendar_action), le nom du Prepare, et la cible du Execute Sub-workflow (`lTsCAxLnJtmS7hXN`). Publier crée une version de rollback.

> ⚠️ **Piège `$json` clobbering** (M4/M5) : le node `Send typing` **remplace `$json`** par la réponse de l'API Telegram. Tous les nodes après `Send typing` lisent le contrat via **`$('Classify the intent').first().json.<champ>`** (nom de node confirmé en live M5), **pas** `$json`.

### B1 — `Is it a Calendar question?` (IF, conditions **String**, combinateur **OR**)
- `{{ $json.intent }}` **is equal to** `calendar_read`
- `{{ $json.intent }}` **is equal to** `calendar_action`
- Entrée branchée sur la sortie **FALSE** de `Is it a Gmail question?`.

> **Décision (D1)** : on route **aussi `calendar_action`** vers M6, mais le sous-wf répond par le message « gated M8 » (N9) sans rien écrire. Ça garde toute la logique agenda au même endroit et donne une réponse propre à Karter. En M8, la branche `action` sera recâblée vers la porte de validation. *(Alternative : ne mettre que `calendar_read` dans l'IF et laisser `calendar_action` filer vers le stub `module pending` — moins propre.)*

### B2 (TRUE) — `Send typing` (Telegram, Send Chat Action)
- Credential `LuminaOsBot - Telegram`. **Chat ID** = `{{ $('Classify the intent').first().json.chat_id }}`, **Action** = `typing`.

### B3 — `Prepare Input for Calendar` (Edit Fields/Set, **Keep Only Set Fields**, mode **Expression**)
- `intent` = `{{ $('Classify the intent').first().json.intent }}`
- `query` = `{{ $('Classify the intent').first().json.query }}`
- `brand` = `{{ $('Classify the intent').first().json.brand }}`
- `chat_id` = `{{ $('Classify the intent').first().json.chat_id }}`
- `message_id` = `{{ $('Classify the intent').first().json.message_id }}`

### B4 — `Run the Calendar ops` (Execute Sub-workflow → `LUMINA-TELEGRAM-CALENDAR-OPS` = **`lTsCAxLnJtmS7hXN`**, **Wait for completion = ON**). Sortie = `{ chat_id, text }`.

### B5 — `Reply to user (calendar)` (Telegram, Send Message)
- Credential `LUMINA-Lyra - Telegram`. **Chat ID** = `{{ $json.chat_id }}`, **Text** = `{{ $json.text }}`.
- **Append n8n Attribution = OFF** · **Parse Mode = None**.

### B6 (FALSE) — la sortie FALSE de `Is it a Calendar question?` file vers **`Is it Maestro?`** (chaîne existante inchangée).

> Méthode résiliente : **dupliquer la branche Gmail** (B2→B5) pour hériter des 2 credentials Telegram correctes (`LuminaOsBot` pour typing, `LUMINA-Lyra` pour l'envoi) et des expressions anti-clobbering, puis ne changer que l'IF, le nom du Prepare et la cible du Execute Sub-workflow.

---

## 5. Recette (test live depuis Telegram, bot Lyra)

| # | Envoi | Attendu |
|---|---|---|
| T1 | « mon agenda aujourd'hui » | « Lyra écrit… » puis événements du jour **regroupés par agenda** (B2C / B2B), heures en clair, fidèles (rien d'inventé). |
| T2 | « mes prochains rendez-vous » | `range=next`, liste chronologique des prochains RDV (les 2 agendas). |
| T3 | « c'est quoi mon planning de demain côté partenaires ? » | `calendar=b2b`, `range=tomorrow`, agenda partners@ uniquement pertinent. |
| T4 | Période sans événement | Message propre « rien de prévu », pas d'erreur brute (grâce à `alwaysOutputData`). |
| T5 | « crée un événement demain 15h avec X » (`calendar_action`) | Message **gated M8** (« pas encore disponible… validation »). **Vérifier : AUCUN événement créé dans les agendas.** |
| T6 | `memory_search`, `web_search`, `gmail_*`, clarification, Maestro | **inchangés** (non-régression M3/M4/M5 + business). |
| T7 | Sortie propre | Aucune erreur `can't parse entities` (Parse Mode None), aucun JSON brut affiché. |

**DoD M6 :** T1–T4 passent ; T5 = zéro écriture (message gated) ; T6 non-régression OK ; T7 sortie propre.

---

## 6. Points à trancher au build

1. ~~**Fuseau horaire**~~ **Tranché (13-07) : `Europe/Paris`**, calcul tz-aware luxon dans N3 (DST géré).
2. **Quel « calendar » exact** pour chaque credential (le node Google Calendar demande de choisir un agenda dans la liste) — primaire du compte (`hello@…` / `partners@…`) sauf si Karter veut un agenda secondaire nommé.
3. **`calendar_action` → M6 (gated) ou stub ?** — D1 le route vers M6 (message gated propre). À rebasculer vers la vraie porte **M8** quand elle sera livrée.
4. **Agenda personnel** (`itiskarter@gmail`) — hors MVP (pas de credential n8n) ; rejoindra le « hub perso » du backlog.
5. **Modèle LLM** — `gpt-5-mini` retenu (cohérent M3–M5). À confirmer.

---

## 7. Pièges à respecter (figés — ne pas les réapprendre)

1. **Lecture seule** en M6 : uniquement **Event : Get Many**. Aucun Create/Update/Delete d'événement (= M8).
2. **Vérifier CHAQUE credential** après import : avec **2 credentials Calendar** (Hello / PARTNERS), l'auto-assignation est **aléatoire** → contrôler node par node (leçon M5 : c'était vrai pour les 2 Gmail).
3. **Set v3.4** : toute valeur `{{ }}` en **mode Expression** (Fixed → texte littéral → bug « chat not found »).
4. **Après un node LangChain** : lire via **`.first()`** (pas `.item`).
5. **Sortie Telegram** : **Attribution OFF** + **Parse Mode None** ; crochets `[..]` pas `<..>`.
6. **`$json` clobbering** après `Send typing` → relire le contrat via `$('Classify the intent').first().json`.
7. **Execute Sub-workflow** : **Wait for completion = ON**. **Execute Workflow Trigger** ⇒ sous-wf **invisible en MCP** (vérif canvas/JSON).
8. **`alwaysOutputData = ON`** sur les 2 lectures Calendar (agenda vide géré).
9. **Google Calendar : Single Events = ON** (sinon les récurrences remontent en règle unique, pas en occurrences datées) + **Order By = startTime**.
10. **Fuseau** : ne pas se fier au « Default Timezone » n8n (noté invalide) → borner et afficher avec un fuseau explicite.
11. **Ne pas empiler** de passes LLM au-delà du nécessaire (latence) : 1 passe comprendre + 1 passe résumer, pas plus.
12. **Construire par import JSON** = méthode résiliente (paste-import ou `file_upload` sur l'input caché « Import from file ») ; éviter les backticks littéraux dans le code des nodes pour rester injectable.

---

## 8. Checklist de build (ordre d'exécution)

1. [ ] P1–P2 : credentials `Hello AFTRSN Calendar` (B2C) + `PARTNERS-AFTRSN Calendar account` (B2B) testées vertes.
2. [ ] P3 : confirmer en live que le Router émet `calendar_read`.
3. [x] P4 : fuseau fixé = `Europe/Paris` (13-07).
4. [ ] Créer `LUMINA-TELEGRAM-CALENDAR-OPS` (Personal, Error Workflow = Sentinel) — **consigner l'ID réel**.
5. [ ] N1 Execute Workflow Trigger (Accept all data).
6. [ ] N2 `Understand request` (gpt-5-mini → plan JSON) + N3 `Parse plan` (calcul bornes ISO).
7. [ ] N4 `Route op` (Switch read/action).
8. [ ] Lecture : N5a/N5b Google Calendar Get Many (B2C + B2B, Time Min/Max, Single Events, alwaysOutputData) + tags + N6 Merge + N7 Summarize + N8 Shape reply.
9. [ ] Action : N9 Shape reply (gated M8).
10. [ ] Gateway Partie B : `Is it a Calendar question?` + `Send typing` + `Prepare Input for Calendar` + `Run the Calendar ops` + `Reply to user (calendar)`, sur la sortie FALSE de `Is it a Gmail question?` (FALSE → `Is it Maestro?`).
11. [ ] Tests §5 en live depuis Telegram (dont : zéro écriture agenda, non-régression M3/M4/M5).
12. [ ] **À validation** → convention documentaire (§9).

---

## 9. Après validation (convention §3 de la passation — automatique)

Produire et **double-sauvegarder** (projet Claude `Marches-a-suivre/` **+** Drive `LUMINA AI DOCS`, dans les **bons sous-dossiers**) :
- `POS-LUMINA_Telegram-Calendar-Ops_<date>.md` (**exact** : IDs réels, noms de nodes, expressions, credentials) — sections **Difficultés / Solutions / Lessons learned**.
- `POS-GENERIQUE_Branche-calendar-ops-telegram.md` (**patron réutilisable**) — mêmes 3 sections.

Puis : **addendum Bible v2.7** (nouveau sous-wf M6 + IDs + invariant « lecture seule, action gatée M8 »), **cocher M6** dans le SPEC-BUILD / la passation, **mettre à jour la mémoire** (`lumina-personal-command-bot`, `lumina-m5-gmail-ops-live` → suite M6).

---

*GUIDE-BUILD M6 v1.0 — 2026-07-13. Credentials Calendar vérifiées en live (Hello AFTRSN Calendar + PARTNERS-AFTRSN Calendar account). Contrats Gateway/Router repris de M5 v1.2 + SPEC-BUILD Telegram v0.3 ; à re-vérifier en live (n8n fait foi). Boussoles : SPEC-BUILD Telegram v0.3 · GUIDE-BUILD M5 v1.2 · Bible 360° v2.5 + v2.6.*
