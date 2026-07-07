---
type: raw
title: "POS-LUMINA_Auto-Ingest-Raw-vers-Wiki_2026-07-07"
source_url: "drive:1kCpnX0h3-UgApDxCgorZabkDFIWa5z0Z"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS-LUMINA — Auto-Ingest `raw/ → wiki/` (multi-coffres) — v2

> **Statut :** construit, testé de bout en bout, **opérationnel sur les 3 coffres** (`ai-automation`, `brands`, `personal`) — 2026-07-07.
> **Remplace** la v1 du 2026-07-06 (mono-coffre).
> **Objet :** automatiser l'unique étape manuelle du pipeline Lumina — la compilation `raw/ → wiki/` — avec le **moteur Claude** (Sonnet), orchestré par **n8n**, en **full automatique**, sur un **runner Claude Code headless** hébergé sur Coolify.
> **Environnement :** n8n `n8n.aftersunpeople.com` · repos GitHub `frescatzi/{ai-automation,brands,personal}` · runner `runner.aftersunpeople.com` (Coolify/Hetzner, projet AFTRSN-Automation).

---

## 1. Place dans le pipeline

```
Drive Inbox → (intake n8n) → GitHub raw/  ──push raw/**──▶  n8n Webhook (LUMINA — Auto-Ingest)
                                                              │  filtre (main + regex raw/) → HTTP POST /ingest {repo, files}
                                                              ▼
                                              Runner Claude headless (Coolify)
                                              1. git reset --hard origin/main      (coffre choisi via {repo})
                                              2. claude -p --model sonnet          (compile UNIQUEMENT les fichiers changés → wiki/, draft)
                                              3. maj index.md + log.md (+ _archive_queue.json en append)
                                              4. git commit + push  →  main
                                                              │
                              push wiki/** ─────┴──▶ pgvector + Notion (workflows existants)
                              archive cron ───────▶ balaie raw/ vers Drive
```

L'auto-ingest est le **chaînon manquant automatisé** entre l'intake (remplit `raw/`) et l'archivage (vide `raw/`).

---

## 2. Prérequis (secrets — gate CEO, jamais dans le chat/repo)

| Secret | Où | Rôle |
|---|---|---|
| `ANTHROPIC_API_KEY` | env Coolify | fait tourner Claude Code (vraie clé `sk-ant-…`) |
| `GIT_PAT` | env Coolify | **token seul** (`github_pat_…`), **Contents: Read and write sur les 3 repos** |
| `INGEST_SECRET` | env Coolify **et** credential/header n8n | authentifie l'appel n8n → runner (mêmes valeurs des 2 côtés) |
| (webhook GitHub, secret optionnel) | GitHub + n8n | non vérifié en HMAC dans cette v1 (sécurité = URL obscure + INGEST_SECRET) |

⚠️ **Coolify passe les env en build-args** → ils apparaissent **en clair dans les logs de déploiement**. Après tout changement, considérer les secrets comme exposés et les **faire tourner**.

---

## 3. Le runner Claude headless (Coolify)

Service HTTP (`GET /health`, `POST /ingest`). Déployé en **« Dockerfile sans Git »** : un Dockerfile auto-suffisant embarque `server.js` + `package.json` + `ingest-prompt.md`. Fichier de référence : projet Claude → `runner-auto-ingest/Dockerfile.standalone`.

**Variables d'env (Coolify) :** `ANTHROPIC_API_KEY`, `GIT_PAT`, `INGEST_SECRET`, `GIT_AUTHOR_NAME=lumina-auto-ingest`, `GIT_AUTHOR_EMAIL=bot@aftersunpeople.com`, `VAULT_DIR=/vault`, `MODEL=sonnet`, `PORT=8080`. Défauts internes : `ALLOWED_REPOS=frescatzi/ai-automation,frescatzi/brands,frescatzi/personal`, `DEFAULT_REPO=frescatzi/ai-automation`.

**Réglages Coolify :** domaine `runner.aftersunpeople.com`, port exposé pris du `EXPOSE 8080`, **pas de volume** sur `/vault` (voir pièges), healthcheck facultatif.

**Endpoint `POST /ingest` (étapes) :** auth header `X-Ingest-Secret` → verrou anti-concurrence → valider `repo` contre l'allowlist → `git reset --hard origin/main` du coffre → `claude -p "$(cat prompt)" --model sonnet --permission-mode bypassPermissions --allowedTools Read Write Edit Bash Grep Glob --max-turns 30` (prompt = base + liste des fichiers changés) → si `git status` non vide : commit + `pull --rebase` + push → renvoie `{status, repo, changed, files, summary}`.

**`GET /health`** renvoie `{"ok":true,"busy":false,"repos":[…3 coffres…]}`.

---

## 4. Le workflow n8n `LUMINA — Auto-Ingest — Raw→Wiki`

`Webhook (POST) → Code (filtre) → HTTP Request (/ingest)`. Fichier de réf : `runner-auto-ingest/n8n-LUMINA-Auto-Ingest.json`.

- **Webhook** : `responseMode: onReceived` (répond 200 tout de suite à GitHub). Production URL = `https://n8n.aftersunpeople.com/webhook/lumina-auto-ingest` (workflow **Active**).
- **Code (filtre anti-boucle)** : ne garde que `ref = refs/heads/main` **ET** au moins un fichier ajouté/modifié dont le chemin matche **`/(^|\/)raw\//` ET finit par `.md`** (capte aussi les sous-dossiers marque `AFTRSN/raw/…`). Ignore les suppressions (archivage) et les commits du bot (qui ne touchent pas `raw/`). Sort `{proceed, files, repo, after}` ou rien.
- **HTTP Request** : `POST https://runner.aftersunpeople.com/ingest`, header `X-Ingest-Secret`, body JSON `{ repo: $json.repo, files: $json.files, reason: 'push '+$json.after }`, timeout 300 s, retry off.

**Webhooks GitHub** : sur **chacun** des 3 repos → Settings → Webhooks → Payload URL = la Production URL n8n · content type `application/json` · event = *push*.

---

## 5. Le prompt d'ingest (`ingest-prompt.md`) — qualité éditoriale

Générique et multi-coffres : il **s'appuie sur le `CLAUDE.md` de chaque coffre** pour les règles propres (structure, marques, périmètre), et impose les garde-fous universels : dédup par empreinte ; synthèse (pas de copier-coller) ; frontmatter `type: wiki, status: draft, publish: none, vault/brand selon le coffre, sources = chemin EXACT, related, updated` ; jamais `active`/`notion` ; ne touche pas `raw/` ; exclut passations/milestones/sessions/specs et le hors-périmètre ; met à jour `index.md` + `log.md` ; **append** (jamais remplacer) `_archive_queue.json`. Le runner ajoute en fin de prompt la liste des fichiers à compiler → Claude **ne traite que ceux-là** (rapide).

---

## 6. Garde-fous automatiques (remplacent la revue humaine)

1. **Dédup par empreinte** → tue les re-captures « même contenu, date différente ».
2. **Anti-boucle** → déclenchement sur `raw/**.md` ajouté/modifié uniquement ; les commits du bot (log.md/wiki) et les suppressions d'archivage n'entraînent rien.
3. **Verrou de concurrence** → un seul ingest à la fois.
4. **Plafond** → `--max-turns 30` + compilation ciblée.
5. **`draft` / `publish: none`** → rien n'atteint agents/Notion sans **promotion manuelle** en `active`+`notion`.
6. **Runner = seul writer de `wiki/`** + `git pull --rebase` avant push.
7. **Allowlist repos** → refuse tout repo hors des 3 coffres.

---

## 7. Pièges rencontrés & solutions (vécus le 06–07/07)

| Piège | Solution |
|---|---|
| Claude Code refuse de tourner en **root** | Utiliser l'utilisateur `node` (UID 1000 déjà dans `node:20-slim`) : `USER node` + `chown -R node:node`. |
| **Volume** monté sur `/vault` (root-only) → node ne peut plus écrire, « not empty », rm échoue | **Pas de volume** : `/vault` éphémère, re-cloné à chaque conteneur. |
| Placeholders collés au lieu des vraies valeurs (`ANTHROPIC_API_KEY`, PAT) | Coller les **vraies** valeurs ; vérifier `debut=sk-ant-…`. |
| PAT sans **Contents: write** → `403 denied` au push (clone/fetch marchent car repo public) | Fine-grained token : *Only select repositories* + **Contents: Read and write** sur les 3 repos. |
| **Oubli de « Save » avant « Deploy »** dans Coolify → rebuild de l'ancien Dockerfile | **Toujours Save avant Deploy.** |
| Changement d'env non pris en compte | **Redeploy** après tout changement de variable. |
| `brands` range les raw sous `<MARQUE>/raw/` (+ doublons de casse `AFTRSN`/`aftrsn`) | Filtre regex **profondeur-agnostique** `/(^|\/)raw\//` ; consolider les doublons de casse. |
| Fichier créé **sans `.md`** ou dans `raw/raw/` (création manuelle GitHub) | Le filtre exige `.md` ; créer depuis la racine ou taper juste le nom dans le bon dossier. |
| Clone partiel bloquant | `ensureRepo` nettoie `/vault` avant de re-cloner. |

---

## 8. Tests de recette (validés)

1. Push d'un `.md` dans `raw/` (ou `<marque>/raw/`) → exécution n8n en ~2 s → runner répond `{"status":"ok","repo":…}` → page wiki (draft) ou ligne log poussée. **brands testé OK** (`repo: frescatzi/brands`).
2. Idempotence : re-pousser le même contenu → aucune nouvelle page (dédup).
3. Anti-boucle : le commit du bot ne redéclenche pas d'ingest (filtre vide).
4. Sécurité : `INGEST_SECRET` faux → `403 forbidden`.

---

## 9. Exploitation & sécurité

- **Rotation des secrets** : après toute exposition (logs Coolify), régénérer le PAT (révoquer l'ancien) + changer `INGEST_SECRET` (Coolify **et** n8n) → **Redeploy**. Valider par un push test (doit répondre `ok`, pas `403`).
- **Promotion** : une page reste `draft` jusqu'à passage manuel `active` + `publish: notion` (seul moment où elle atteint les agents).
- **Nettoyage** : les fichiers de test dans `raw/` seront balayés par le cron d'archivage.

---

## 10. Évolutions prévues

- **Promotion assistée** : workflow qui passe une page `draft → active` + `publish: notion` sur demande.
- **Schedule de secours** (horaire) au cas où un webhook est manqué.
- **Revue optionnelle** : basculer l'écriture sur une branche `auto-ingest` + PR si un filet humain devient souhaitable.
- **HMAC GitHub** : vérifier la signature du webhook dans n8n.

---

*POS-LUMINA — Auto-Ingest raw→wiki (multi-coffres) — v2 — 2026-07-07. Patron générique : voir `POS-GENERIQUE_Runner-LLM-headless-declenche-par-webhook`. Double sauvegarde : projet Claude « Claude Lessons » + Google Drive (règle CLAUDE.md).*
