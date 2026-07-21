---
type: raw
title: "POS-GENERIQUE_Bot-Telegram-prive-auth-ACL-Postgres"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_bot-telegram-prive-auth-acl-postgres.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — Bot Telegram privé + auth par ACL Postgres cloisonnée (GÉNÉRIQUE)

**Version :** v1.0
**Date :** 2026-07-08
**Projet :** Patron réutilisable (toute marque / tout bot privé)
**Étape couverte :** Prérequis terrain d'un bot Telegram privé mono-owner (extensible multi-rôles)

> Paramètres : `<MARQUE>`, `<BOT_NAME>`, `<BOT_USERNAME>`, `<TOKEN>`, `<OWNER_CHAT_ID>`, `<ACL_DB>`, `<ACL_USER>`, `<ACL_PWD>`, `<ACL_CRED_NAME>`, `<TELEGRAM_CRED_NAME>`, `<PG_INSTANCE>`.

---

## 1. Objectif

Poser les prérequis d'un bot Telegram **privé** : bot dédié, token en credential, `chat_id` de l'owner, et **authentification par table ACL dans une base Postgres séparée et cloisonnée** (lecture seule). Approche à privilégier quand la fonctionnalité *Variables* n8n (`$vars`) n'est pas disponible (pas de licence) et/ou quand on veut isoler l'auth des données métier. Cette table est aussi la **fondation d'une AAL/RBAC** multi-rôles.

## 2. Prérequis

- Accès **@BotFather**.
- Accès **n8n** + à l'orchestrateur d'hébergement (ex. Coolify) exposant un **terminal** sur le conteneur Postgres.
- Une **instance Postgres** `<PG_INSTANCE>` joignable par n8n (réseau partagé) + un **compte admin** Postgres.

## 3. Procédure pas à pas

1. **Créer le bot.** @BotFather → `/newbot` → nom `<BOT_NAME>` → username `<BOT_USERNAME>` (finit par `bot`) → récupérer `<TOKEN>`.
2. **Stocker le token.** n8n → Credentials → **Telegram API** → coller `<TOKEN>` → nommer `<TELEGRAM_CRED_NAME>`.
3. **Récupérer le `chat_id`.** Envoyer un message au bot, puis `https://api.telegram.org/bot<TOKEN>/getUpdates` → `message.chat.id` = `<OWNER_CHAT_ID>` (**valeur numérique**, ≠ username du bot). Refermer l'onglet.
4. **Auth par ACL Postgres cloisonnée :**
   1. Terminal du conteneur Postgres → `psql -U <admin> -d postgres` (invite `postgres=#`).
   2. `CREATE DATABASE <ACL_DB>;` (**seul**, hors transaction).
   3. `CREATE USER <ACL_USER> WITH PASSWORD '<ACL_PWD>';` puis `GRANT CONNECT ON DATABASE <ACL_DB> TO <ACL_USER>;` (**mot de passe entre guillemets simples**).
   4. `\c <ACL_DB>` (**ligne isolée** ; invite → `<ACL_DB>=#`).
   5. `CREATE TABLE bot_acl (chat_id TEXT PRIMARY KEY, role TEXT NOT NULL DEFAULT 'owner', note TEXT);`
   6. `INSERT INTO bot_acl (chat_id, role, note) VALUES ('<OWNER_CHAT_ID>', 'owner', '<MARQUE>');`
   7. `GRANT USAGE ON SCHEMA public TO <ACL_USER>;` + `GRANT SELECT ON bot_acl TO <ACL_USER>;`
   8. Credential n8n Postgres `<ACL_CRED_NAME>` (Host/Port/SSL de l'instance ; Database `<ACL_DB>` ; User `<ACL_USER>` ; Password `<ACL_PWD>`). Test connection.

## 4. Vérification / recette

- `SELECT has_table_privilege('<ACL_USER>','bot_acl','SELECT');` → `t`.
- Node n8n Postgres (credential `<ACL_CRED_NAME>`) : `SELECT role FROM bot_acl WHERE chat_id='<OWNER_CHAT_ID>';` → `owner`.

## 5. Difficultés rencontrées (typiques)

1. Fonctionnalité *Variables* n8n indisponible sans licence.
2. Risque de mettre l'auth dans la même base que les données métier (blast radius).
3. Terminal de conteneur qui **entrelace les collages multi-lignes** → `\c` parasité (`invalid integer value … for connection option "port"`).
4. Mot de passe **non quoté** → *syntax error* → user absent → `GRANT` en échec (`role does not exist`).
5. Confusion **shell vs psql** (`sh: CREATE: not found`).

## 6. Solutions implémentées

1. Auth via **base Postgres séparée** + **user lecture seule dédié** + **credential dédiée**.
2. **Moindre privilège** : le user n'a que `SELECT` sur `bot_acl`.
3. SQL **ligne par ligne**, jamais en bloc collé.
4. `\c` **seul sur sa ligne**.
5. Mot de passe **toujours** entre `'...'`.
6. Vérifier l'invite (`…=#` = psql ; `#` = shell) avant chaque commande.
7. `CREATE DATABASE` exécuté **seul**.

## 7. Lessons learned

1. Sans `$vars`, préférer une **table ACL Postgres** (vs `$env`) : cloisonnable, auditable, extensible en AAL/RBAC.
2. **L'enforcement d'accès se fait au niveau data** (base/collection interrogeable), **jamais dans le seul prompt LLM** (prompt-injection).
3. **Allow list** (présence de ligne) ≠ **ACL** (colonne `role`) : deux dimensions, une table.
4. `chat_id` = **numérique**, ≠ username du bot.
5. Secrets (token, mot de passe DB) → **uniquement en credential**, jamais dans doc/log/prompt.
6. Extension multi-rôles : ajouter colonnes `role` (member/guest), `brand_scope`, `expires_at` ; le node d'auth renvoie le `role` + un `scope` ; le filtrage confidentiel se fait en forçant `collection`/`brand` selon le rôle.

## 8. Références

- Patron du node d'auth : `SELECT role FROM bot_acl WHERE chat_id = <input.chat_id>` → autorisé si ligne renvoyée ; sinon message « Accès non autorisé ».
- POS exact jumeau (valeurs réelles) : `POS-LUMINA_Bot-Telegram-dedie-et-auth-ACL_2026-07-08.md`.

---

*POS-GÉNÉRIQUE v1.0 — 2026-07-08. Double-sauvegardé : projet Claude + Google Drive « LUMINA AI DOCS » (POS-GENERIQUE).*
