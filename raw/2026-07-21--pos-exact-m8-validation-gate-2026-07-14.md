---
type: raw
title: "POS-EXACT_M8-Validation-Gate_2026-07-14"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_m8-validation-gate_2026-07-14.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-EXACT — M8 · Validation Gate (porte d'approbation à boutons inline Telegram)

**Date :** 2026-07-14 · **Projet :** LUMINA Personal Assistant Bot « Lyra » (Telegram) · **Module :** M8
**Objet :** garde-fou central d'approbation humaine. Toute action critique (à ce stade : **envoi d'email réel**) ne s'exécute **jamais** automatiquement. Lyra affiche un récap + deux boutons **[✅ Valider] / [❌ Annuler]**, mémorise la demande en attente dans la table `pending_action`, et n'exécute qu'après un clic explicite de l'owner. C'est la **première capacité d'envoi réel** du bot — uniquement via la porte.
**Statut :** ✅ **FAIT, PUBLIÉ & VALIDÉ BOUT-EN-BOUT** (brouillon → carte 2 boutons → clic Valider → **email réellement reçu**).
**Boussoles :** `SPEC-BUILD_LUMINA-TELEGRAM-GATEWAY-INTENT-ROUTER` (v0.3) + `BIBLE_LUMINA-OS_360` (v2.5 → addenda → v2.9). Réfère au GUIDE-BUILD M8 v0.1 et à la passation `PASSATION_LUMINA-Telegram-Lyra_M8-fait-doc-et-M9_2026-07-14.md`.

---

## 1. Décisions de cadrage (validées Karter, 2026-07-14)

1. **UX = boutons inline Telegram** `[✅ Valider] / [❌ Annuler]` sous le récap (pas de commande à taper). Les commandes texte `/validate | /cancel` restent acceptées en repli (intent `validation_response` déjà au Router), mais l'ergonomie par défaut = boutons.
2. **État en attente = table dédiée `pending_action`** (pas de réemploi de `conversation_state` : séparation propre clarification / approbation).
3. **Périmètre câblé en v1 = envoi mail seulement.** La table + la porte sont génériques : ajouter une action = un `action_type` + une branche d'exécuteur (calendar_create/update, delete, workflow_modify, memory_write, spend, publish à l'usage).
4. **Expiration = 1 heure**, paresseuse (vérifiée au clic, pas de cron).
5. **Owner-only** : seul `chat_id = 776345147` (`role='owner'` dans `bot_acl`) peut valider/annuler ; un clic d'un autre compte = rejet + message « Non autorisé ».

---

## 2. IDs & contrats réels (n8n fait foi — 2026-07-14, re-vérifier via store Pinia avant modif)

| Brique | ID / réf | Rôle en M8 |
|---|---|---|
| **Gateway** `LUMINA-TELEGRAM-GATEWAY` | `uGF3pSv6aTK79cqi` (Publié, **59 nodes** après M8) | Reçoit `message` **et** `callback_query` ; héberge register + resolve |
| **Intent-Router** | `KTLwQi7ZKHDTexMZ` | Marque déjà `requires_validation` + `validation_response` (aucune modif) |
| **M5 Gmail Ops** (+ branche draft M7) | `LUCkYnz7GhjkyCtZ` (actif) | Produit le brouillon ; **expose `mailbox / to / subject / body`** au payload |
| **M6 Calendar Ops** | `lTsCAxLnJtmS7hXN` (publié) | Écriture d'événement à gater (patron prévu, pas câblé v1) |
| **AFTRSN-Maestro** | `OlrQO21u178SjgBK` | **Modèle basculé sur OpenAI `gpt-5-mini`** (node `OpenAI Chat Model (test M)`, cred « OpenAi account ») — cf. §6. Confirmé live 14-07 16:52. |
| **DB écriture** | `LUMINA_Postgres` | Table `pending_action` (+ `conversation_state`) |
| **DB ACL (lecture)** | `lumina_access.bot_acl` · user `luminabot` · cred `LUMINA_UserAccess_Postgres` | Auth owner du clic callback |
| **Credential Telegram** | `LUMINA-Lyra - Telegram` | sendMessage + inline keyboard + answerCallbackQuery + editMessageText |
| **Credentials Gmail** | `HELLO AFTRSN GMAIL` (B2C hello@) / `PARTNERS-AFTRSN GMAIL` (B2B partners@) | Envoi réel gaté (`message:send`) |
| **Error Workflow (Sentinel)** | `xzaH0uWy0idKVphF` | Branché sur les workflows touchés |
| **Owner** | `chat_id = 776345147` | Seul autorisé à valider/annuler |

---

## 3. Modèle de données — table `pending_action` (DB `LUMINA_Postgres`)

```sql
CREATE TABLE IF NOT EXISTS pending_action (
  id          BIGSERIAL PRIMARY KEY,
  chat_id     TEXT        NOT NULL,
  action_type TEXT        NOT NULL,     -- gmail_send (v1) | calendar_create | calendar_update | delete | workflow_modify | memory_write | spend | publish
  payload     JSONB       NOT NULL,     -- de quoi exécuter : mailbox, to, subject, body…
  summary     TEXT        NOT NULL,     -- récap humain affiché dans Telegram
  status      TEXT        NOT NULL DEFAULT 'pending',   -- pending | done | cancelled | expired
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT now() + interval '1 hour',
  decided_at  TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS idx_pending_action_chat_status ON pending_action (chat_id, status);
```

**Invariants :**
- L'exécution ne lit **que** `payload` (jamais un état volatil du chat) → une demande reste exécutable après d'autres messages.
- Expiration **paresseuse** (`expires_at > now()` au clic), pas de cron pour le MVP.
- `status` : `pending → done | cancelled | expired`, jamais en arrière. Clic **idempotent** : `UPDATE … WHERE status='pending'` — 0 ligne = déjà traité/expiré.

---

## 4. Architecture réelle — deux ajouts **additifs** dans la Gateway (chemin message préservé)

Le Telegram Trigger écoute désormais `["message","callback_query"]`. Un IF `M8 - Is callback?` splitte tout de suite en entrée :
- **message** → flux normal existant (clarification → memory-search → web/ops → Gmail → Calendar → Maestro → Reply) — inchangé ;
- **callback_query** → branche **resolve** (§4.B).

### 4.A Branche « register » (enregistrer une demande)
Insérée **après** `Run the Gmail ops` (retour de M5 avec la branche draft M7) :

```
Run the Gmail ops  (retour : requires_validation, mailbox, to, subject, body)
  → M8 - Needs validation?   (IF : requires_validation == true  ET  mailbox NON vide)   ← condition DOUBLE
        ├─ TRUE →
        │    M8 - Register pending_action   (Postgres INSERT … RETURNING id)
        │      → M8 - Send validation card  (Telegram sendMessage + inline keyboard)
        │            texte  = summary
        │            boutons= [✅ Valider] cb "m8:v:<id>"  |  [❌ Annuler] cb "m8:c:<id>"
        │      → FIN (aucun envoi)
        └─ FALSE → flux normal (pas de carte)
```

> **Point critique** : la carte ne s'affiche **QUE si un brouillon a réellement été créé** — d'où le test **`requires_validation == true` ET `mailbox` non vide**. Sans le second terme, un brouillon échoué (mailbox vide) afficherait une **carte fantôme** pointant sur rien. Cf. §7 Lessons.

### 4.B Branche « resolve » (résoudre un clic)

```
Telegram Trigger (["message","callback_query"])
  → M8 - Is callback?
     ├─ message → flux normal
     └─ callback →
          Parse callback     (Code : décision v|c + id ; chat_id = callback.message.chat.id ; from_id = callback.from.id)
          → Auth callback (ACL)   (Postgres scalar-subquery : role de bot_acl WHERE chat_id = from_id)
          → Owner?           (IF role == 'owner')
             ├─ non → Answer not authorized  (answerCallbackQuery « Non autorisé ») → FIN
             └─ oui → Is validate?   (IF decision == 'v')
                 ├─ VALIDER →
                 │    Claim row  (Postgres UPDATE pending_action SET status='done', decided_at=now()
                 │                WHERE id=$1 AND status='pending' AND expires_at>now() RETURNING *)
                 │      ├─ 0 ligne → Answer expired (« ⏱ Expiré ou déjà traité ») → FIN
                 │      └─ 1 ligne → Is B2B?  (IF mailbox == partners@ / brand B2B)
                 │             ├─ Send B2C  (Gmail message:send, cred HELLO)   → Answer sent + Edit (sent)
                 │             └─ Send B2B  (Gmail message:send, cred PARTNERS) → Answer sent + Edit (sent)
                 └─ ANNULER →
                      Postgres UPDATE … SET status='cancelled', decided_at=now() WHERE id=$1 AND status='pending'
                      → Answer cancelled + Edit (cancelled)  (retire les boutons)
```

**Points n8n réels :**
- **`answerCallbackQuery`** (resource `callback`, op `answerQuery`, `queryId`) obligatoire pour éteindre le spinner du bouton, **même credential bot** (`LUMINA-Lyra - Telegram`).
- **`editMessageText`** après décision pour retirer/figer les boutons → empêche le double-clic visuel.
- **Parse Mode = None**, crochets `[..]` (piège figé sortie Telegram).
- Postgres : `operation:'executeQuery'`, 1 statement paramétré `$1..$n`, `options.queryReplacement` en tableau, **scalar-subquery** pour toujours renvoyer 1 ligne à l'auth ; **credential écriture (`LUMINA_Postgres`) ≠ credential ACL (`LUMINA_UserAccess_Postgres`)**.

---

## 5. Exécuteur mail (v1) — `gmail_send`

- Le node Gmail n'a **pas** d'opération « send draft » → on utilise **`message:send`** avec `to / subject / body` tirés du `payload` (exposés par M5/M7).
- **`appendAttribution:false`** → retire le footer « This email was sent with n8n ».
- Routage mailbox : `Is B2B?` → `Send B2C` (cred **HELLO**, hello@) ou `Send B2B` (cred **PARTNERS**, partners@).
- `summary` = `📧 Prêt à envoyer — [B2C hello@] À: <to> · Objet: <subject>`.

> **Invariant** : M5/M7 = **brouillon uniquement** ; M8 est l'**unique** chemin d'envoi réel, sur ordre explicite + clic owner.

---

## 6. Chantier FIABILITÉ rédaction (résolu le 14-07, documenté ici car pré-requis de M8)

**Symptôme :** la rédaction M7 échouait (`Draft OK?`=false, body vide) → message d'erreur au lieu d'un brouillon.

**Diagnostic :** **Maestro sous Claude Haiku 4.5 « narrate-and-stop »** — il restituait le récit de sa réconciliation (« Calling Call_LUMINA-AI-Router… Marketing RÉFUTÉ… escalade en attente de Karter ») au lieu du JSON final `{subject,body}`. Le durcissement du brief M7 + un **retry 1×** n'ont **pas** suffi (échec aux 2 essais) : blocage systématique (Marketing = Claude Sonnet refuse la formule simple cycle après cycle ; Haiku narre au lieu de clore).

**Fix appliqué & validé :** **bascule du modèle de l'orchestrateur Maestro `OlrQO21u178SjgBK`** : Anthropic **Haiku 4.5 → OpenAI `gpt-5-mini`** (node actif = `OpenAI Chat Model (test M)`, cred « OpenAi account » ; node `Anthropic Chat Model` conservé **débranché** = revert 1 geste). **Résultat : le brouillon sort du 1ᵉʳ coup.** Confirmé live le 14-07 (Maestro `updatedAt` 16:52, node `OpenAI Chat Model (test M)` = `lmChatOpenAi`). Claude Sonnet reste OK **en sous-agent** (Marketing).

**Aussi en service :**
- Brief M7 durci (règle de **clôture obligatoire** = JSON final non vide même après réfutation).
- **Retry 1×** conservé : `Draft OK?` false → `Prepare Maestro brief (retry)` → `Draft via Maestro 2` → `Parse draft 2` → `Draft OK? 2` → **`Draft chosen`** (convergence). `Shape reply (draft)` lit `$('Draft chosen')`.
- **Strip RGPD déterministe** dans `Draft chosen` : filtrage ligne-à-ligne retirant le bloc confidentialité (regex `right to access | privacy policy | politique de confidentialit | confidentialite | aftersunpeople.com/confidentialite | donn.es personnelles`), en gardant salutation + signature.

---

## 7. Difficultés rencontrées / Solutions implémentées / Lessons learned

### Difficultés rencontrées
1. **Carte de validation fantôme** : le premier câblage déclenchait la carte sur `requires_validation` seul → une carte s'affichait même quand le brouillon avait échoué (mailbox vide, rien à envoyer).
2. **Rédaction non aboutie** : Maestro (Haiku 4.5) narrait sa boucle au lieu de renvoyer le JSON → `Draft OK?`=false, aucun brouillon ; non réglé par brief durci + retry.
3. **Publish n8n en échec « Missing required credential »** sur le node Gmail créé via le store Pinia.
4. **Footer n8n + bloc RGPD** parasitaient les mails.
5. **Callbacks Telegram muets** au premier essai (spinner qui tourne).

### Solutions implémentées
1. **Condition double** `M8 - Needs validation?` = `requires_validation == true` **ET** `mailbox` non vide → la porte se déclenche sur le **succès de l'action préparée**, pas sur l'intention.
2. **Bascule modèle orchestrateur** Maestro Haiku 4.5 → **gpt-5-mini** (+ retry 1× + garde-fou `Draft OK?`/`Draft chosen`).
3. Node Gmail via store : ajouter **`parameters.authentication:'oAuth2'`** en plus de `credentials:{gmailOAuth2:{id,name}}`.
4. `appendAttribution:false` + **strip RGPD** ligne-à-ligne dans `Draft chosen`.
5. **`answerCallbackQuery`** systématique + **`editMessageText`**, **même credential bot** que l'envoi.

### Lessons learned (pièges figés)
- **Une porte de validation se déclenche sur le SUCCÈS de l'action préparée, pas sur l'intention.** Toujours tester l'artefact réel (`mailbox`/`draft_id` non vide), pas seulement le flag d'intention → sinon carte fantôme.
- **Orchestrateur : le MODÈLE compte plus que le prompt.** **Claude Haiku 4.5 en orchestrateur = « narrate-and-stop »** (restitue sa boucle au lieu du livrable) — non réglé par brief durci + retry. Fix = un **modèle qui obéit au format** (gpt-5-mini). Toujours prévoir un **garde-fou déterministe en aval** (IF succès/échec, aucune action sur sortie non conforme). Claude Sonnet reste bon **en sous-agent**.
- **Node Gmail via store Pinia** : exige `parameters.authentication:'oAuth2'` sinon Publish échoue. Le node Gmail n'a **pas** de « send draft » → utiliser **`message:send`**.
- **Telegram callback** : `answerCallbackQuery` + `editMessageText` avec **le même credential bot** que l'envoi, sinon retour dans le vide ; Parse Mode None, crochets `[..]`.
- **Postgres (n8n 2.6)** : scalar-subquery pour toujours renvoyer 1 ligne à l'auth ; clic idempotent via `WHERE status='pending'` ; credential écriture ≠ credential ACL.
- **Édition Gateway via store Pinia** puis **salir par un VRAI drag sur un node réel** → **Publish** orange → Publier → **recharger + re-vérifier via le store** (l'état publié fait foi ; `stateIsDirty` non fiable).

---

## 8. Tests de recette (résultats réels)

| # | Scénario | Attendu | Résultat |
|---|---|---|---|
| T1 | « envoie un mail à X … » | Brouillon + **carte 2 boutons** ; **aucun envoi** | ✅ |
| T2 | Clic **✅ Valider** (owner) | Mail **envoyé** (email reçu) ; spinner éteint ; message édité ; `status=done` | ✅ **email réellement reçu** |
| T3 | Clic **❌ Annuler** (owner) | Aucun envoi ; message édité ; `status=cancelled` | ✅ |
| T4 | Valider **> 1 h** | « ⏱ Expiré ou déjà traité » ; aucun envoi | ✅ |
| T5 | Double-clic Valider | 2ᵉ clic sans effet (idempotence) | ✅ |
| T6 | Clic depuis un autre compte | « Non autorisé » ; aucune action | ✅ |
| T8 | Non-régression M3→M7 | Mémoire / web / Gmail lecture / brouillon inchangés | ✅ |

**DoD atteint** : brouillon → carte → Valider → **email reçu**. Annuler / expiration / double-clic / autre compte gérés.

---

## 9. Reste de ménage (non bloquant)

- Supprimer le(s) brouillon(s) de test dans `hello@` + éventuels épisodes/leçons de test dans les banques `aftrsn`.
- Scratch `hjf8vp32FtZpKQmd` (« My workflow 2 ») à supprimer.
- Décider : garder le node `Anthropic Chat Model` débranché (revert) ou le retirer ; éventuellement renommer `OpenAI Chat Model (test M)` → `OpenAI Chat Model`.

---

## 10. Périmètre incrémental (mêmes rails)

La table + register/resolve sont **génériques**. Ajouter une action = un `action_type` + une branche d'exécuteur dans le resolve : **écriture d'événement M6**, **suppression** (Drive/event/mémoire), **dépense / publication**, **modif workflow/mémoire**. Aucun changement de schéma.

---

*POS-EXACT M8 — 2026-07-14. Première capacité d'envoi réel du bot, uniquement via la porte, approbation humaine explicite. n8n fait foi : re-vérifier les IDs/contrats en live (store Pinia) avant toute modif. Boussoles : le Spec Build Lumina Telegram + le Lumina Bible 360.*
