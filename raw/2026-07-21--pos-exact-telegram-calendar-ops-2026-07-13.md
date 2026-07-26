---
type: raw
title: "POS-EXACT_Telegram-Calendar-Ops_2026-07-13"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_telegram-calendar-ops_2026-07-13.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — LUMINA-TELEGRAM · M6 CALENDAR OPS (valeurs réelles)

**Date :** 2026-07-13
**Statut :** ✅ Construit, testé en isolé, câblé dans la Gateway, **validé en live depuis Telegram** (agenda du jour B2C/B2B, heure de Paris).
**Projet :** LUMINA OS · Personal Assistant Bot « Lyra » (@luminaLyraBot)
**Objet :** brancher l'intent `calendar_read` (+ `calendar_action` gaté) sur un sous-workflow Calendar dédié — **lecture seule** des 2 agendas business After Sun People.

---

## 1. IDs et objets réels

| Élément | Valeur réelle |
|---|---|
| Sous-workflow M6 | `LUMINA-TELEGRAM-CALENDAR-OPS` = **`lTsCAxLnJtmS7hXN`** (projet Personal, Publié) |
| Gateway | `LUMINA-TELEGRAM-GATEWAY` = **`uGF3pSv6aTK79cqi`** (34 → **39 nodes** après M6, Publié) |
| Credential Calendar B2C | **`HELLO-AFTRSN Calendar`** (id `sMZcyjIFDnYq2xs5`, Google Calendar OAuth2 API) → agenda `hello@aftersunpeople.com` |
| Credential Calendar B2B | **`PARTNERS-AFTRSN Calendar`** (id `jzQDFFWKmFHcykO7`) → agenda `partners@aftersunpeople.com` |
| Modèle LLM | `gpt-5-mini` via credential **`OpenAi account`** (id `fqQvWAsn0tBEhSGU`) |
| Error Workflow (Sentinel) | `LUMINA-SENTINEL/ERROR-WATCH` = `xzaH0uWy0idKVphF` |
| Credentials Telegram (branche Gateway) | **`LUMINA-Lyra - Telegram`** pour **Send typing ET Reply** (fix 13-07 : le typing était sur `LuminaOsBot`, ce qui affichait l'indicateur dans le chat LUMINA OS au lieu de Lyra — corrigé sur les 3 branches Gmail/Web/Calendar) |
| Fuseau | **Europe/Paris** (calcul luxon dans Parse plan) |

---

## 2. Sous-workflow `LUMINA-TELEGRAM-CALENDAR-OPS` (14 nodes)

Flux :
```
Receive from Gateway → Understand request (gpt-5-mini) → Parse plan → Route op (Switch read/action)
   ├─ read   → [Cal B2C + Cal B2B (Event:Get Many)] → Tag b2c / Tag b2b → Merge agendas → Summarize → Shape reply (read)
   └─ action → Shape reply (gated)   ← message « pas encore dispo, validation M8 », AUCUNE écriture
```

Contrat : entrée `{ intent, query, brand, chat_id, message_id }` → sortie `{ chat_id, text }`.

**Node Google Calendar (`n8n-nodes-base.googleCalendar` v1.3) — paramètres exacts (Cal B2C / Cal B2B) :**
```
resource = event ; operation = getAll
calendar = { __rl:true, mode:"list", value:"hello@aftersunpeople.com" (resp. partners@) }
returnAll = false ; limit = {{ $('Parse plan').first().json.max }}
timeMin  = {{ $('Parse plan').first().json.time_min }}     ← TOP-LEVEL (pas sous options)
timeMax  = {{ $('Parse plan').first().json.time_max }}     ← TOP-LEVEL
options.orderBy = startTime
options.recurringEventHandling = expand      ← (= « All Occurrences », requis par orderBy=startTime)
alwaysOutputData = ON
credential = HELLO-AFTRSN Calendar (resp. PARTNERS-AFTRSN Calendar)
```

**`Parse plan` (Code, calcul luxon Europe/Paris)** — extrait clé :
```javascript
const ZONE = 'Europe/Paris';
const now = DateTime.now().setZone(ZONE);
let timeMin = now, timeMax = now.plus({ days });
if (range === 'today')    { timeMin = now.startOf('day'); timeMax = now.endOf('day'); }
if (range === 'tomorrow') { const t = now.plus({ days: 1 }); timeMin = t.startOf('day'); timeMax = t.endOf('day'); }
if (range === 'week')     { timeMin = now; timeMax = now.plus({ days: 7 }); }
if (range === 'next')     { timeMin = now; timeMax = now.plus({ days }); }
// -> time_min/time_max en ISO via timeMin.toISO()
```

Le JSON complet importable est archivé : `LUMINA-TELEGRAM-CALENDAR-OPS.json` (racine projet Claude).

---

## 3. Branche M6 dans la Gateway (Partie B)

Insertion sur la **sortie FALSE de `Is it a Gmail question?`** (qui pointait vers `Is it Maestro?`) :

```
Is it a Gmail question? (FALSE) → Is it a Calendar question? (IF, OR)
        conditions : {{ $('Classify the intent').first().json.intent }} == calendar_read
                     {{ $('Classify the intent').first().json.intent }} == calendar_action
   ├─ TRUE  → Send typing (calendar) [cred LUMINA-Lyra] → Prepare Input for Calendar
              → Run the Calendar ops (Execute Sub-wf lTsCAxLnJtmS7hXN, Wait ON) → Reply to user (calendar) [cred LUMINA-Lyra]
   └─ FALSE → Is it Maestro?   (chaîne existante inchangée)
```

Les nodes `Send typing (calendar)`, `Prepare Input for Calendar`, `Reply to user (calendar)` sont des **clones exacts** des nodes Gmail équivalents (mêmes expressions `$('Classify the intent')…`, mêmes credentials Telegram) → seuls l'IF (conditions), le nom du Prepare et la cible du Run changent.

---

## 4. Difficultés rencontrées

1. **Structure JSON du node Google Calendar inconnue au départ** : ma 1ʳᵉ version mettait `timeMin/timeMax` sous `options` → ignorés (les champs After/Before restaient aux défauts `$now`/`$now.plus({week:1})`), et `calendar` en mode « By ID » avec `primary` → refusé (« Not a valid Google Calendar ID »).
2. **Un node perdu à l'import + duplication canvas galère** : lors d'une 1ʳᵉ construction, un node Calendar a disparu (édition parasite → auto-reconnexion), et la duplication manuelle + câblage au pixel a échoué (drop-sur-connexion, positions).
3. **Éditeur d'expression n8n auto-ferme `{{ }}` `()"'`** → taper une expression complète produit des doublons (`{{ … }} }}`). Idem pour le mock data du trigger.
4. **Multi-select du canvas KO via l'automatisation** : ni shift-clic ni cmd-clic n'accumulent la sélection ; le rubber-band est erratique → impossible de copier proprement la branche Gmail (5 nodes) au canvas.
5. **PERSISTANCE (faux positif live)** : après avoir câblé la branche en manipulant le store Pinia, la publication a **promu un draft resté à 34 nodes** → le 1ᵉʳ test Telegram est tombé sur le stub « module en construction ».

---

## 5. Solutions implémentées

1. **Récupérer la structure exacte du node via le store Pinia** (`$pinia._s → workflows → workflow.nodes`) après l'avoir configuré une fois à la main → `timeMin/timeMax` sont **top-level**, `options.recurringEventHandling:"expand"`, `calendar` en mode **list** avec l'email. Reconstruire un JSON correct et **réimporter** (plus fiable que réparer).
2. **Import JSON avec `credentials:{<type>:{id,name}}` embarqués** → auto-liés à l'import (zéro triangle rouge, zéro assignation manuelle). IDs de credentials lus dans le store.
3. **Remplir les champs/expressions/mock via `document.execCommand('insertText', false, texte)`** (insertion en bloc, pas d'auto-fermeture). Pour les champs UI simples, taper l'expression **sans les `}}` finaux**.
4. **Câbler la Gateway en manipulant le store** : cloner les nodes de la branche Gmail (`JSON.parse(JSON.stringify)`, `id=crypto.randomUUID()`, positions offset), ajuster (conditions IF, `workflowId.value`), puis `wf.setNodes(...)` + `wf.setConnections(...)` → le canvas se re-render depuis le store. Validé d'abord sur un **workflow scratch**.
5. **Forcer l'autosave avant de publier** : après la mutation store, **déclencher un vrai événement canvas** (léger drag d'un node, ~20px) → l'autosave écrit le draft (39 nodes) sur le serveur (`ui.stateIsDirty` → `false`), **puis** publier. **Recharger + re-vérifier via le store** = seule preuve fiable.

---

## 6. Lessons learned (pièges figés)

- **n8n Google Calendar Get Many** : `timeMin`/`timeMax` sont **top-level** ; `calendar` en **From list** (l'alias `primary` est refusé par la validation) ; `recurringEventHandling:"expand"` requis si `orderBy:startTime` ; `alwaysOutputData` ON pour gérer l'agenda vide.
- **Le store Pinia de n8n est la source de vérité fiable** pour lire (IDs, params, credentials, connexions) ET écrire (`setNodes`/`setConnections`) quand le canvas ne coopère pas — mais **ça n'affiche que**, ça ne **sauvegarde pas** tout seul.
- **Persistance = un vrai événement UI** : un changement programmatique doit être suivi d'une interaction canvas réelle (drag) pour déclencher l'autosave du draft, sinon Publish promeut l'ancien état. **Toujours recharger et re-vérifier.**
- **Import avec credentials embarqués** = gain de temps énorme vs assignation manuelle (surtout avec 2 credentials du même type où l'auto-assignation est aléatoire).
- **`execCommand('insertText', ...)` = l'arme anti-auto-fermeture** des éditeurs CodeMirror de n8n.
- **Lecture seule = défaut MVP** ; création/modif d'événement (surtout avec invités) = action critique → **autorisée sur ordre explicite de Karter**, via la validation **M8**.
- **Le `Send typing` doit utiliser la MÊME credential bot que le `Reply` (Lyra)** : sinon l'indicateur « typing » (sendChatAction) part dans le chat de l'autre bot (LuminaOsBot / « LUMINA OS ») au lieu du chat Lyra. Bug hérité du clone Gmail, corrigé le 13-07 sur les 3 branches (Gmail, Web, Calendar) → toutes sur `LUMINA-Lyra - Telegram`.

---

## 7. Recette live validée (Telegram, bot Lyra)

- « mon agenda aujourd'hui » → **OK** : « Agenda du jour — 13/07/2026 (heure de Paris) », B2C hello@ / B2B partners@, « Aucun événement pour aujourd'hui » (agendas vides, géré proprement).
- « mes prochains rendez-vous » (range=next) → testé en isolé OK.
- « crée un événement… » (`calendar_action`) → message gated M8, **zéro node d'agenda exécuté**.
- Non-régression M3/M4/M5 : chaîne inchangée en amont.

---

*POS-EXACT M6 — 2026-07-13. Double-sauvegardé : projet Claude « Claude Lessons » (`Marches-a-suivre/`) + Google Drive `LUMINA AI DOCS/POS-EXACT`. Voir aussi POS-GENERIQUE_Branche-calendar-ops-telegram + addendum Bible v2.7.*
