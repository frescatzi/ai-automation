---
type: raw
title: "GUIDE-BUILD_LUMINA-M8-Validation-Gate_2026-07-14"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m8-validation-gate_2026-07-14.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# 🔧 GUIDE-BUILD — LUMINA M8 · Validation Gate (porte d'approbation `/validate`)

**Version :** v0.1 (fiche de cadrage, à confirmer avant build)
**Date :** 2026-07-14 · **Projet :** LUMINA OS · Personal Assistant Bot « Lyra » (Telegram) · **Module :** M8
**Boussoles :** `SPEC-BUILD_LUMINA-TELEGRAM-GATEWAY-INTENT-ROUTER` (v0.3) + `BIBLE_LUMINA-OS_360` (v2.5 → addenda v2.8).
**Réfère à :** passation `PASSATION_LUMINA-Telegram-Lyra_M8-et-suite_2026-07-14.md`.

> **Portée** : introduire le **garde-fou central d'approbation humaine**. Toute action critique (envoi mail, écriture/création d'événement, suppression, dépense, publication, modif workflow/mémoire) ne s'exécute **jamais** automatiquement : Lyra affiche un récap + deux boutons **[✅ Valider] / [❌ Annuler]**, mémorise la demande en attente, et n'exécute qu'après un clic explicite de l'owner. C'est le pré-requis pour **activer l'envoi réel** en M5/M7 et l'**écriture d'événement** en M6.

---

## 1. Décisions de cadrage (validées Karter, 2026-07-14)

1. **UX = boutons inline Telegram** `[✅ Valider] / [❌ Annuler]` sous le récap (pas de commande à taper). Les commandes texte `/validate | /cancel` restent acceptées en repli (l'intent `validation_response` existe déjà au Router), mais l'ergonomie par défaut = boutons.
2. **État en attente = nouvelle table `pending_action`** (dédiée), pas de réemploi de `conversation_state` (séparation propre clarification / approbation).
3. **Périmètre gaté dès v1** : envoi mail (M5/M7) · écriture/création d'événement (M6) · suppression · modif workflow/mémoire · dépense · publication.
4. **Expiration = 1 heure** : passé ce délai la demande expire, l'action est refusée (récap marqué expiré).
5. **Owner-only** : seul `chat_id = 776345147` (`role='owner'` dans `bot_acl`) peut valider/annuler. Un clic d'un autre compte = rejet silencieux + log.

---

## 2. IDs & contrats réels (n8n fait foi — re-vérifier via store Pinia avant câblage)

| Brique | ID / réf | Rôle en M8 |
|---|---|---|
| **Gateway** `LUMINA-TELEGRAM-GATEWAY` | `uGF3pSv6aTK79cqi` (Publié, 39 nodes) | Reçoit `message` **et** (nouveau) `callback_query` ; héberge la porte |
| **Intent-Router** | `KTLwQi7ZKHDTexMZ` | Marque déjà `requires_validation` + `validation_response` (aucune modif requise) |
| **M5 Gmail Ops** (+ branche draft M7) | `LUCkYnz7GhjkyCtZ` (v5, actif) | Produit le brouillon Gmail ; **fournit le `draft_id` + mailbox** à gater |
| **M6 Calendar Ops** | `lTsCAxLnJtmS7hXN` (publié) | Écriture d'événement à gater (create/update) |
| **AFTRSN-Maestro** | `OlrQO21u178SjgBK` (Haiku 4.5) | Inchangé (zone rouge) |
| **DB écriture** | `LUMINA_Postgres` | Accueille la table `pending_action` |
| **DB ACL (lecture)** | `lumina_access.bot_acl` / user `luminabot` / cred `LUMINA_UserAccess_Postgres` | Auth owner du clic callback |
| **Credential Telegram (envoi)** | `LUMINA-Lyra - Telegram` | Send message + boutons + answerCallbackQuery + editMessage |
| **Credentials Gmail** | `HELLO-AFTRSN GMAIL` (B2C) / `PARTNERS-AFTRSN GMAIL` (B2B) | Envoi réel gaté |
| **Credentials Calendar** | `HELLO-AFTRSN Calendar` `sMZcyjIFDnYq2xs5` / `PARTNERS-AFTRSN Calendar` `jzQDFFWKmFHcykO7` | Écriture d'événement gatée |
| **Error Workflow (Sentinel)** | `xzaH0uWy0idKVphF` | À laisser branché sur les workflows touchés |

---

## 3. Modèle de données — table `pending_action` (DB `LUMINA_Postgres`)

```sql
CREATE TABLE IF NOT EXISTS pending_action (
  id          BIGSERIAL PRIMARY KEY,
  chat_id     TEXT        NOT NULL,
  action_type TEXT        NOT NULL,      -- gmail_send | calendar_create | calendar_update | delete | workflow_modify | memory_write | spend | publish
  payload     JSONB       NOT NULL,      -- tout ce qu'il faut pour exécuter (mailbox, draft_id, to, subject, event fields…)
  summary     TEXT        NOT NULL,      -- récap humain affiché dans Telegram
  status      TEXT        NOT NULL DEFAULT 'pending',   -- pending | done | cancelled | expired
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT now() + interval '1 hour',
  decided_at  TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS idx_pending_action_chat_status ON pending_action (chat_id, status);
```

**Invariants** :
- L'exécution ne lit **que** `payload` (jamais un état volatil du chat) → une demande reste exécutable même après d'autres messages.
- Expiration **paresseuse** (vérifiée au moment du clic via `expires_at > now()`), pas besoin de cron pour le MVP ; un sweep de nettoyage est un raffinement optionnel.
- `status` passe `pending → done|cancelled|expired`, jamais en arrière. Le clic est **idempotent** : `UPDATE … WHERE id=$1 AND status='pending'` — 0 ligne affectée = déjà traité/expiré.

---

## 4. Architecture — deux ajouts dans la Gateway

### 4.A Branche « ENREGISTRER une demande » (register)
Insérée là où une action critique est prête (ex. après la création du brouillon dans la branche draft de M5/M7, ou avant une écriture d'événement M6).

```
[action prête + requires_validation=true]
  → Register pending_action (Postgres INSERT … RETURNING id)
  → Send validation card (Telegram sendMessage + inline keyboard)
        texte  = summary
        boutons = [✅ Valider]=cb "m8:v:<id>"  |  [❌ Annuler]=cb "m8:c:<id>"
  → FIN (aucune exécution)
```

> **callback_data** limité à 64 octets → format compact `m8:v:<id>` / `m8:c:<id>` (l'id entier tient largement).

### 4.B Branche « RÉSOUDRE un clic » (resolve)
Le Telegram Trigger doit désormais inclure l'update **`callback_query`**.

```
Telegram Trigger (message + callback_query)
  → IF update is callback_query ?
     └─ OUI →
        Parse callback (Code : extrait decision v|c + id ; chat_id = callback.message.chat.id ; from_id = callback.from.id)
        → Auth owner (Postgres SELECT role FROM bot_acl WHERE chat_id=$1)  [from_id]
             └─ pas owner → answerCallbackQuery « Non autorisé » → FIN
        → IF decision == v ?
           ├─ VALIDER →
           │    Claim row (Postgres UPDATE pending_action SET status='done', decided_at=now()
           │                WHERE id=$1 AND status='pending' AND expires_at>now() RETURNING *)
           │      ├─ 0 ligne → answerCallbackQuery « ⏱ Expiré ou déjà traité » + editMessage → FIN
           │      └─ 1 ligne → Switch action_type :
           │            gmail_send      → Gmail: send draft (mailbox du payload)  → answer «✅ Envoyé» + editMessage
           │            calendar_create → Calendar: create event (payload)        → answer «✅ Créé»   + editMessage
           │            calendar_update → Calendar: update event                  → answer «✅ Mis à jour»
           │            (delete/workflow_modify/memory_write/spend/publish : même patron, exécuteur dédié)
           └─ ANNULER →
                Postgres UPDATE … SET status='cancelled', decided_at=now() WHERE id=$1 AND status='pending'
                → answerCallbackQuery «❌ Annulé» + editMessage (retire les boutons)
```

**Points n8n importants** :
- **Answer Callback Query** (opération Telegram `answerCallbackQuery`) obligatoire pour éteindre le spinner du bouton, avec le **même credential bot** que l'envoi (`LUMINA-Lyra - Telegram`) — sinon le retour part dans le vide (rappel piège M6 : `Send typing` doit partager le credential du `Reply`).
- **Edit Message Text** (ou Edit Reply Markup) pour retirer les boutons après décision → empêche un double-clic.
- **Parse Mode = None**, crochets `[..]` pas `<..>` (piège figé sortie Telegram).
- Requête Postgres **1 statement paramétré**, `queryReplacement` en tableau, credential écriture (`LUMINA_Postgres`) ≠ credential ACL (`LUMINA_UserAccess_Postgres`).

---

## 5. Exécuteur mail (priorité v1) — `gmail_send`

Réutilise le **brouillon déjà créé** par M5/M7 (source unique de vérité, pas de re-rédaction) :
- À l'enregistrement, la branche draft renvoie `draft_id` + `mailbox` (`hello@` B2C / `partners@` B2B) → stockés dans `payload`.
- À la validation, node **Gmail → « Send Draft »** (ou `messages.send` avec le draft) sur la bonne credential mailbox.
- `summary` = `📧 Prêt à envoyer — [B2C hello@] À: <to> · Objet: <subject>`.

> **Invariant** : tant que M8 n'est pas validé live, **aucun envoi réel n'est activé**. Le brouillon reste l'état par défaut ; M8 est l'unique chemin d'envoi.

---

## 6. Périmètre v1 vs incrémental

| Action | v1 (ce build) | Notes |
|---|---|---|
| **gmail_send** (M5/M7) | ✅ câblé + testé | Priorité n°1 (déblocage attendu) |
| **calendar_create / calendar_update** (M6) | ✅ patron câblé | M6 renvoyait déjà l'action gatée |
| **delete** (Drive/event/mémoire) | ⏳ patron prévu, exécuteur branché à l'usage | Action destructive : même porte |
| **workflow_modify / memory_write / spend / publish** | ⏳ patron prévu | Enregistrés via `action_type`, exécuteur ajouté au cas par cas |

La **table + la porte (register/resolve) sont génériques** : ajouter une action = un `action_type` + une branche d'exécuteur dans le Switch. Aucun changement de schéma.

---

## 7. Tests de recette

| # | Scénario | Attendu |
|---|---|---|
| T1 | « envoie un mail à X … » | Brouillon créé (M7) + **carte de validation** avec 2 boutons ; **aucun envoi** |
| T2 | Clic **✅ Valider** (owner) | Mail **envoyé** ; spinner éteint ; message édité «✅ Envoyé» ; row `status=done` |
| T3 | Clic **❌ Annuler** (owner) | **Aucun** envoi ; message édité «❌ Annulé» ; row `status=cancelled` |
| T4 | Valider **après > 1 h** | «⏱ Expiré ou déjà traité » ; aucun envoi ; row `status` reste ≠ done |
| T5 | Double-clic Valider | 2ᵉ clic sans effet (idempotence : 0 ligne affectée) |
| T6 | Clic depuis un autre compte | « Non autorisé » ; aucune action |
| T7 | Écriture d'événement M6 sur ordre explicite | Carte de validation → Valider → event créé |
| T8 | Non-régression M3→M7 | Mémoire / web / Gmail lecture / brouillon inchangés |

**DoD** : T1–T6 + T8 verts (T7 si l'exécuteur calendar est câblé dans ce build).

---

## 8. Méthode d'édition (résiliente — rappel des pièges figés)

- Édition Gateway via **store Pinia** dans l'onglet n8n piloté : lire `wf.workflow.nodes/.connections`, cloner, muter, `wf.setNodes()/setConnections()`, puis **salir par un VRAI drag** d'un node (événement canvas *trusted*) → bouton **« Publish »** orange → Publier → **reload + re-vérif store** (l'état publié fait foi ; `stateIsDirty` non fiable).
- **Injection JS = base64/ASCII** (`atob`), sources sans accents, backticks via `String.fromCharCode` ; **restreindre les retours** (filtre anti-secret « Cookie/query string data »).
- **Telegram Trigger** : ajouter `callback_query` aux Updates (sinon les clics ne déclenchent rien).
- **DDL `pending_action`** : exécuter d'abord sur un node Postgres scratch (valider), pas directement sur le chemin critique.
- **Multi-select canvas KO en auto** → câbler connexion par connexion via le store.

---

## 9. Ordre de build

1. [ ] Créer la table `pending_action` (DDL §3) sur `LUMINA_Postgres` (via node Postgres scratch, vérifier).
2. [ ] Gateway : Telegram Trigger → ajouter update `callback_query`.
3. [ ] Gateway : brancher la **branche resolve** (Parse callback → Auth owner → IF v/c → claim/switch → Gmail send + answer + edit).
4. [ ] Brancher la **branche register** après la création du brouillon (M5/M7) : INSERT `pending_action` + envoi de la carte à boutons ; remplacer le « brouillon prêt » terminal par la carte.
5. [ ] Exécuteur `gmail_send` (Send Draft sur bonne mailbox).
6. [ ] (option) Exécuteur `calendar_create/update` (M6).
7. [ ] Publier (Gateway) ; tester T1–T8 depuis Telegram ; non-régression M3→M7.
8. [ ] Doc convention §4 : POS-EXACT + POS-GENERIQUE (Difficultés / Solutions / Lessons), double-save Claude + Drive, addendum **Bible v2.9**, cocher Spec Build, MàJ mémoire (`lumina-m8-validation-gate`).

---

## 10. Difficultés rencontrées / Solutions / Lessons learned
*(à compléter pendant le build — sections obligatoires par convention §4)*

- **Difficultés rencontrées** : —
- **Solutions implémentées** : —
- **Lessons learned** : —

---

*GUIDE-BUILD M8 v0.1 — 2026-07-14. Design verrouillé sur les 5 décisions §1. n8n fait foi : re-vérifier les IDs/contrats en live (store Pinia) avant de câbler. À confirmer par Karter avant mutation de la Gateway de production.*
