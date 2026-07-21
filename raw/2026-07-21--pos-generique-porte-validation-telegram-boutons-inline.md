---
type: raw
title: "POS-GENERIQUE_Porte-validation-Telegram-boutons-inline"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_porte-validation-telegram-boutons-inline.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GENERIQUE — Porte de validation humaine à boutons inline (bot Telegram + n8n)

**Objet :** patron réutilisable pour introduire un **garde-fou d'approbation humaine** avant toute **action critique** exécutée par un bot conversationnel (envoi mail, écriture/suppression d'événement, dépense, publication, modif de workflow/mémoire). L'action n'est **jamais** automatique : le bot affiche un récap + deux boutons **[✅ Valider] / [❌ Annuler]**, persiste la demande, et n'exécute qu'après un clic explicite d'un utilisateur autorisé.
**Pré-requis :** bot Telegram privé avec Gateway n8n + table ACL (owner/roles). Base Postgres accessible en écriture.

---

## 1. Principe

Séparer **préparer** de **exécuter**. Une action critique est d'abord **préparée** (brouillon, payload d'événement, ordre de suppression…) puis **enregistrée** en attente ; l'exécution réelle est déclenchée par un **clic de bouton** (callback Telegram) authentifié, idempotent et à expiration.

Deux ajouts **additifs** dans la Gateway (le chemin message existant reste intact) :
- **register** : après qu'une action critique est prête → INSERT en base + envoi d'une **carte** à boutons ; fin (aucune exécution).
- **resolve** : le Trigger écoute aussi `callback_query` → un clic authentifié verrouille la ligne et lance l'exécuteur.

---

## 2. Modèle de données — table d'actions en attente

```sql
CREATE TABLE IF NOT EXISTS pending_action (
  id          BIGSERIAL PRIMARY KEY,
  chat_id     TEXT        NOT NULL,
  action_type TEXT        NOT NULL,     -- typologie : <action>_send | <action>_create | delete | spend | publish | …
  payload     JSONB       NOT NULL,     -- tout ce qu'il faut pour exécuter (jamais un état volatil du chat)
  summary     TEXT        NOT NULL,     -- récap humain affiché
  status      TEXT        NOT NULL DEFAULT 'pending',   -- pending | done | cancelled | expired
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT now() + interval '1 hour',
  decided_at  TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS idx_pending_action_chat_status ON pending_action (chat_id, status);
```

**Invariants :**
- L'exécuteur ne lit **que** `payload` → la demande reste valide même après d'autres messages.
- Expiration **paresseuse** (vérifiée au clic via `expires_at > now()`) — pas de cron pour un MVP.
- `status` avance seulement (`pending → done | cancelled | expired`). Clic **idempotent** : `UPDATE … WHERE status='pending'` (RETURNING) — 0 ligne = déjà traité/expiré.
- Table + porte **génériques** : ajouter une action = un nouveau `action_type` + une branche d'exécuteur. Aucun changement de schéma.

---

## 3. Branche « register » (enregistrer)

À placer **là où l'action critique est prête et réussie** :

```
[action préparée : renvoie un flag "à valider" + les champs d'exécution]
  → IF  "à valider" == true  ET  <artefact réel non vide>      ← condition DOUBLE (cf. piège n°1)
        ├─ TRUE →
        │    INSERT pending_action (chat_id, action_type, payload, summary) RETURNING id
        │    → sendMessage + inline keyboard :
        │          texte  = summary
        │          boutons= [✅ Valider] callback_data "v:<id>"  |  [❌ Annuler] callback_data "c:<id>"
        │    → FIN (aucune exécution)
        └─ FALSE → flux normal (pas de carte)
```

> `callback_data` ≤ 64 octets → format compact `v:<id>` / `c:<id>`.

---

## 4. Branche « resolve » (résoudre un clic)

Le Telegram Trigger doit inclure l'update **`callback_query`** (sinon les clics ne déclenchent rien). Un IF en entrée splitte `message` (flux normal) vs `callback_query` (résolution).

```
Trigger (["message","callback_query"])
  → Is callback ?
     ├─ message → flux normal
     └─ callback →
          Parse callback   (Code : decision v|c + id ; from_id = callback.from.id)
          → Auth (ACL)     (Postgres scalar-subquery : role WHERE chat_id = from_id)
          → Owner/role OK ?
             ├─ non → answerCallbackQuery « Non autorisé » → FIN
             └─ oui → Is validate ? (decision == v)
                 ├─ VALIDER →
                 │    Claim  (UPDATE … SET status='done', decided_at=now()
                 │            WHERE id=$1 AND status='pending' AND expires_at>now() RETURNING *)
                 │      ├─ 0 ligne → answerCallbackQuery « ⏱ Expiré ou déjà traité » → FIN
                 │      └─ 1 ligne → Switch action_type → exécuteur dédié
                 │                    → answerCallbackQuery « ✅ Fait » + editMessageText
                 └─ ANNULER →
                      UPDATE … SET status='cancelled', decided_at=now() WHERE id=$1 AND status='pending'
                      → answerCallbackQuery « ❌ Annulé » + editMessageText (retire les boutons)
```

**Obligatoire :**
- **`answerCallbackQuery`** pour éteindre le spinner du bouton — **même credential bot** que l'envoi.
- **`editMessageText`** (ou editReplyMarkup) après décision → fige/retire les boutons (anti double-clic).
- Parse Mode = None, crochets `[..]`.
- Postgres : 1 statement paramétré, `queryReplacement` en tableau, **scalar-subquery** pour toujours renvoyer 1 ligne à l'auth ; **credential écriture ≠ credential ACL**.

---

## 5. Exécuteur (patron)

- Réutiliser autant que possible l'artefact déjà préparé ; sinon reconstruire depuis `payload` (source unique de vérité).
- Un `action_type` = une branche. Ajouter une capacité = ajouter une branche au Switch, sans toucher au schéma ni à la porte.
- Après exécution : `answerCallbackQuery` de confirmation + `editMessageText` récap final.

---

## 6. Difficultés rencontrées / Solutions / Lessons learned

### Difficultés rencontrées
1. **Carte fantôme** : la porte se déclenchait sur l'intention, donc une carte apparaissait même quand l'action préparée avait échoué (artefact vide).
2. **Livrable de l'orchestrateur non conforme** : un LLM orchestrateur « raconte » sa boucle au lieu de renvoyer le format attendu → l'action préparée est vide.
3. **Publish n8n en échec** sur un node d'action (credential) édité programmatiquement.
4. **Callbacks muets** (spinner qui tourne) au premier essai.

### Solutions implémentées
1. **Condition double** au register : flag d'intention **ET** artefact réel non vide → la porte se déclenche sur le **succès** de l'action préparée.
2. **Choisir un modèle d'orchestrateur qui respecte le format de sortie** (un modèle « obéissant » plutôt qu'un raisonneur bavard) + **garde-fou déterministe** en aval (IF succès/échec, aucune action sur sortie non conforme) + retry borné.
3. Éditer le node d'action programmatiquement en incluant **explicitement le champ d'authentification** attendu par le node (pas seulement l'objet credential).
4. **`answerCallbackQuery` + `editMessageText` systématiques**, **même credential bot** que l'envoi.

### Lessons learned (pièges figés)
- **Une porte de validation se déclenche sur le SUCCÈS de l'action préparée, pas sur l'intention.** Tester l'artefact réel, pas le flag seul.
- **En orchestration, le MODÈLE compte plus que le prompt.** Un modèle qui « narre » ne clôt pas ; préférer un modèle qui obéit au format et poser un **garde-fou déterministe** en aval. Un raisonneur bavard peut rester excellent **en sous-agent**.
- **Action critique = jamais automatique** : lecture + préparation OK ; envoi / écriture / suppression / dépense / publication = **gaté**, sur ordre explicite + clic autorisé.
- **Clic idempotent + expiration paresseuse** : `WHERE status='pending'` + `expires_at>now()` suffisent, pas besoin de cron pour un MVP.
- **Callback Telegram** : `answerCallbackQuery` + `editMessageText` avec le credential bot, sinon retour dans le vide.
- **Édition de workflow programmatique** : après mutation, **salir par une vraie interaction** avant de publier, puis **recharger et re-vérifier l'état publié** (ne pas se fier à un flag « dirty »).

---

## 7. Tests de recette (gabarit)

| # | Scénario | Attendu |
|---|---|---|
| T1 | Demande d'action critique | Action préparée + **carte 2 boutons** ; **aucune exécution** |
| T2 | Clic Valider (autorisé) | Action **exécutée** ; spinner éteint ; message édité ; `status=done` |
| T3 | Clic Annuler | Aucune exécution ; message édité ; `status=cancelled` |
| T4 | Valider après expiration | « Expiré ou déjà traité » ; aucune exécution |
| T5 | Double-clic Valider | 2ᵉ clic sans effet (idempotence) |
| T6 | Clic non autorisé | « Non autorisé » ; aucune action |
| T7 | Non-régression | Le chemin message existant inchangé |

---

*POS-GENERIQUE — Porte de validation humaine à boutons inline. Patron dérivé du build LUMINA M8 (Telegram « Lyra »). Rejouable sur toute action critique et toute marque : nouvelle capacité = nouveau `action_type` + branche d'exécuteur.*
