---
type: raw
title: "POS-LUMINA_Bot-Telegram-dedie-et-auth-ACL_2026-07-08"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_bot-telegram-dedie-et-auth-acl_2026-07-08.md"
captured: 2026-07-21
vault: brands
brand: LUMINA
immutable: true
---

# POS — Bot Telegram dédié + auth par ACL Postgres cloisonnée (EXACT)

**Version :** v1.0
**Date :** 2026-07-08
**Projet :** LUMINA OS · Personal Assistant Bot
**Étape couverte :** P1–P4 (Prérequis terrain, Vague 1 du SPEC-BUILD)

---

## 1. Objectif

Mettre en place les prérequis terrain du Assistant Bot : (a) créer le bot Telegram dédié, (b) stocker son token en credential n8n, (c) récupérer le `chat_id` de l'owner, (d) mettre en place le mécanisme d'authentification. **Décision d'implémentation** : faute de licence n8n (fonctionnalité *Variables* / `$vars` indisponible), l'auth ne passe pas par une variable mais par une **table ACL dans une base Postgres séparée et cloisonnée**, lue en lecture seule — ce qui pose aussi la fondation de la future AAL/RBAC.

## 2. Prérequis

- Accès à **@BotFather** (Telegram).
- Accès **n8n** (`https://n8n.aftersunpeople.com:5678`) et **Coolify** (stack « N8N WITH POSTGRES AND WORKER »).
- Instance Postgres du stack : **`pgvector/pgvector:pg16`**, service `Postgresql`, *Running (healthy)*, jointe à n8n via le réseau prédéfini du stack.
- Compte admin Postgres (env `POSTGRES_USER` / `POSTGRES_PASSWORD` du stack).
- **Pas** de licence n8n → pas de *Variables* `$vars`.

## 3. Procédure pas à pas

1. **P1 — Créer le bot dédié.** @BotFather → `/newbot` → nom **« LUMINA Command »** → username **`luminaLyraBot`** (unique, finit par `bot`). Récupérer le **token** (jamais collé ailleurs qu'en credential).
2. **P2 — Stocker le token.** n8n → Credentials → **Telegram API** → coller le token → nommer **`LUMINA-Lyra - Telegram`** → Save.
3. **P3 — Récupérer le `chat_id`.** Envoyer un message au bot dans Telegram, puis ouvrir `https://api.telegram.org/bot<token>/getUpdates` → lire `message.chat.id` = **`776345147`** (identifiant numérique de Karter). Refermer l'onglet (l'URL contient le token).
4. **P4 — Auth par ACL Postgres cloisonnée.**
   1. Coolify → onglet **Terminal** → conteneur **Postgresql** → ouvrir psql :
      ```bash
      psql -U postgres -d postgres
      ```
      (invite attendue : `postgres=#`).
   2. Créer la base **seule** (hors transaction) :
      ```sql
      CREATE DATABASE lumina_access;
      ```
   3. Créer le user lecture seule (**mot de passe entre guillemets simples**) :
      ```sql
      CREATE USER luminabot WITH PASSWORD '<MDP_LUMINABOT>';
      GRANT CONNECT ON DATABASE lumina_access TO luminabot;
      ```
   4. Se reconnecter à la base cible (**ligne isolée**, invite → `lumina_access=#`) :
      ```
      \c lumina_access
      ```
   5. Créer la table ACL + insérer la ligne owner :
      ```sql
      CREATE TABLE bot_acl (chat_id TEXT PRIMARY KEY, role TEXT NOT NULL DEFAULT 'owner', note TEXT);
      INSERT INTO bot_acl (chat_id, role, note) VALUES ('776345147', 'owner', 'Karter');
      ```
   6. Donner le moindre privilège au user :
      ```sql
      GRANT USAGE ON SCHEMA public TO luminabot;
      GRANT SELECT ON bot_acl TO luminabot;
      ```
   7. Créer la credential n8n **`LUMINA_UserAccess_Postgres`** (Host/Port/SSL recopiés de `LUMINA_Postgres` — même conteneur ; Database `lumina_access` ; User `luminabot` ; Password `<MDP_LUMINABOT>`). Test connection → vert.

## 4. Vérification / recette

- `SELECT rolname FROM pg_roles WHERE rolname='luminabot';` → renvoie `luminabot`.
- `SELECT has_table_privilege('luminabot','bot_acl','SELECT');` → `t`.
- Node n8n **Postgres → Select** (credential `LUMINA_UserAccess_Postgres`, table `bot_acl`) → renvoie `776345147 | owner | Karter`.
- DoD : le SELECT via `luminabot` renvoie `owner`.

## 5. Difficultés rencontrées

1. **Pas de licence n8n** → la fonctionnalité *Variables* (`$vars.KARTER_CHAT_ID`) prévue au SPEC-BUILD n'existe pas.
2. Souci de **protection des données** : mettre l'auth dans `LUMINA_Postgres` (base de connaissance ayant déjà subi credential corrompue + TRUNCATE désamorcés) = risque. D'où le choix d'isoler.
3. **Copier-coller multi-lignes dans le terminal du conteneur** = lignes entrelacées → le `\c lumina_access` a été parasité (`invalid integer value "bot_acl" for connection option "port"`), laissant la session sur `postgres`.
4. **Mot de passe sans guillemets** (`CREATE USER ... WITH PASSWORD p4ss...`) → *syntax error* → user non créé → tous les `GRANT` suivants échouent (`role does not exist`).
5. **Confusion shell vs psql** : après une sortie involontaire de psql, le SQL tapé dans `sh` renvoie `sh: 1: CREATE: not found`.

## 6. Solutions implémentées

1. Auth déplacée vers une **base Postgres séparée `lumina_access`** + user **lecture seule `luminabot`** + credential dédiée **`LUMINA_UserAccess_Postgres`**. (Fallback `$env` Coolify écarté : moins cloisonné, et la table ACL est la graine de l'AAL.)
2. **Cloisonnement** : `luminabot` n'a que `SELECT` sur `bot_acl` → la credential du bot ne peut pas lire la base de connaissance ni le pgvector.
3. Exécuter le SQL **ligne par ligne**, jamais en bloc collé, dans ce terminal.
4. `\c` toujours **seul sur sa ligne**.
5. **Mot de passe toujours entre guillemets simples** : `'...'`.
6. Contrôler l'invite avant de taper : `postgres=#` / `lumina_access=#` = psql ; `#` seul = shell.
7. `CREATE DATABASE` exécuté **seul** (ne passe pas en transaction).

## 7. Lessons learned

1. **Sans licence n8n, remplacer `$vars` par une source lisible** : soit `$env` (env Coolify), soit une **table Postgres**. La table gagne : cloisonnable, auditable, et déjà la structure de l'AAL/RBAC (colonnes `role`, `brand_scope`, `expires_at` à venir).
2. **Isoler l'auth = moindre privilège.** Base + user dédiés en lecture seule : un bot compromis ne peut pas atteindre les données métier. **Principe sécu clé (à graver) : l'enforcement d'accès se fait au niveau data (quelle base/collection est interrogeable), jamais uniquement dans le prompt LLM** (risque prompt-injection).
3. Le **terminal de conteneur Coolify mélange les collages multi-lignes** → DDL au coup par coup, `\c` isolé.
4. **Allow list ≠ ACL** : la présence d'une ligne = allow list (AAL) ; la colonne `role` = ACL (niveau d'accès). Même table, deux dimensions.
5. **`chat_id` = identifiant numérique** (ex. `776345147`), à ne pas confondre avec le **username du bot** (`luminaLyraBot`).
6. **Écart vs SPEC-BUILD v0.1** : le node N2 passe de `$vars.KARTER_CHAT_ID` à `SELECT role FROM bot_acl WHERE chat_id=…` via `LUMINA_UserAccess_Postgres`. *n8n fait foi* → SPEC-BUILD et Bible à mettre à jour.
7. **Ne jamais écrire le mot de passe `luminabot` ni le token dans un doc/log/prompt** — ils vivent uniquement en credential. (Mot de passe exposé en clair pendant le build → à roter, cf. liste rotation secrets.)

## 8. Références

- `SPEC-BUILD_LUMINA-TELEGRAM-GATEWAY-INTENT-ROUTER_2026-07-08.md` §2 (P1–P5), §5.1 (node N2), §7 (sécurité).
- `BIBLE_LUMINA-OS_360_2026-07-02.md` (v1.3) — à mettre à jour (addendum auth ACL).
- Bot : **`luminaLyraBot`** · Credential Telegram : **`LUMINA-Lyra - Telegram`**.
- Owner `chat_id` : **`776345147`**.
- Base : **`lumina_access`** · Table : **`bot_acl`** · User : **`luminabot`** (SELECT only).
- Credential Postgres auth : **`LUMINA_UserAccess_Postgres`**.
- Instance : `pgvector/pgvector:pg16`, stack Coolify « N8N WITH POSTGRES AND WORKER ».
- POS générique jumeau : `POS-GENERIQUE_Bot-Telegram-prive-auth-ACL-Postgres.md`.

---

*POS-EXACT v1.0 — 2026-07-08. Double-sauvegardé : projet Claude « Claude Lessons » + Google Drive « LUMINA AI DOCS » (POS-EXACT).*
