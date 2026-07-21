---
type: raw
title: "POS-GENERIQUE_Gateway-Telegram-auth-dispatch-n8n"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_gateway-telegram-auth-dispatch-n8n.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — Gateway Telegram (auth + dispatch) · GÉNÉRIQUE

**Version :** v1.0
**Date :** 2026-07-08
**Projet :** patron réutilisable (dérivé de LUMINA M1 GATEWAY / bot Lyra)
**Étape couverte :** construire une gateway Telegram n8n qui authentifie, traite `/help`, appelle un routeur d'intention et répond.

> Transposable à tout bot `<BOT>` / toute marque. Paramètres : `<BOT_CRED>` (credential Telegram du bot que l'utilisateur contacte), `<ACL_CRED>`, `<ACL_TABLE>`, `<ROUTER_WF>`, `<OWNER_ID>`.

---

## 1. Objectif

Recevoir un message Telegram, **authentifier** l'expéditeur au niveau data (ACL Postgres), router les commandes système (`/help`), déléguer la classification à un **sous-workflow routeur**, et répondre (réel ou stub). Aucun secret dans un node/prompt/log.

## 2. Prérequis
1. Un routeur d'intention `<ROUTER_WF>` (sous-workflow) déjà construit.
2. Le **bot Telegram `<BOT>`** créé, sa credential `<BOT_CRED>` en place (token du bon bot).
3. Table ACL `<ACL_TABLE>(chat_id, role, …)` + credential lecture seule `<ACL_CRED>`.

## 3. Procédure pas à pas (étapes à partir de 1)

1. **Trigger** : Telegram Trigger, *On Message*, **credential = `<BOT_CRED>`** (le bot que l'utilisateur contacte — un seul trigger par bot).
2. **Normalize** (Code, *Run Once for All Items*) : extraire `chat_id, message_id, input, mode(command|natural), command, args` → `return [{ json:{…} }]`.
3. **Auth ACL** (Postgres *Execute Query*, credential `<ACL_CRED>`) — requête **toujours 1 ligne** :
   ```sql
   SELECT (SELECT role FROM <ACL_TABLE> WHERE chat_id = $1) AS role;
   ```
   paramètre `{{ $json.chat_id }}` ; **On Error = Stop Workflow** (fail-safe).
4. **IF autorisé** : `{{ $json.role }}` *is not empty*. FALSE → message « Accès non autorisé » (Chat ID via `{{ $('Normalize').item.json.chat_id }}`).
5. **IF /help** (branche TRUE) : `{{ $('Normalize').item.json.command }}` = `/help` → TRUE : envoyer l'aide.
6. **Set « Contrat Router »** (branche FALSE de /help) : reconstruire le contrat attendu par `<ROUTER_WF>` depuis `$('Normalize')` (+ `user`).
7. **Execute Sub-workflow** → `<ROUTER_WF>`, *Wait for completion*. Sortie = contrat routeur (avec `chat_id` repropagé).
8. **IF clarification** : `{{ $json.needs_clarification }}` *is true* → question de clarification ; sinon → réponse (réelle si le module existe, sinon **stub**).
9. **Mise en service** : dépingler les données de test, **Save + Active** ; vérifier `getWebhookInfo` (`url` non vide).

## 4. Vérification / recette
Depuis Telegram (bot `<BOT>`) : `/help` → aide ; commande claire → intention détectée (réelle/stub) ; message ambigu → clarification ; charabia → unknown. Refus : depuis un compte hors ACL → « Accès non autorisé ».

## 5. Difficultés rencontrées (typiques)
1. Code node : « A 'json' property isn't an object » (Mode ≠ forme de retour).
2. Postgres : 0 ligne quand non autorisé → item perdu ; options « Continue On Fail » déplacées en *Settings → On Error* (selon version).
3. Telegram « chat not found » : nouveau bot sans conversation ouverte côté user.
4. Pas de réaction en live : `getWebhookInfo url:""` → workflow non actif **ou** trigger sur la mauvaise credential/bot.
5. Contexte perdu après le node Auth (`$json` = `{role}`).

## 6. Solutions implémentées
1. Mode **Run Once for All Items** avec `return [{json}]`.
2. Requête **toujours 1 ligne** `SELECT (SELECT role …) AS role` + **On Error=Stop**.
3. `/start` du bot côté user avant tout envoi.
4. Bonne **credential = bon bot** sur le trigger ; **activer** le workflow (webhook posé) ; ne pas test-listen en actif.
5. Repropager le contexte via `$('Normalize')` ; reconstruire le contrat routeur dans un Set.

## 7. Lessons learned
1. **Trigger = bot que l'utilisateur contacte** ; un seul trigger par bot ; ne pas réutiliser un bot d'alertes.
2. **Webhook prod = à l'activation** ; diagnostiquer avec `getWebhookInfo`.
3. **Nouveau bot ⇒ /start** obligatoire avant envoi.
4. **Mode du Code node** cohérent avec la forme de retour.
5. **Auth fail-safe au niveau data** (requête 1-ligne + Stop on error).
6. **Après un node qui écrase `$json`**, référencer les nodes amont par nom.

## 8. Références
- Instanciation concrète : `POS-LUMINA_Telegram-Gateway_2026-07-08.md`.
- Routeur associé : `POS-GENERIQUE_Router-intention-LLM-n8n.md`.
- Auth : `POS-GENERIQUE_Bot-Telegram-prive-auth-ACL-Postgres.md`.
