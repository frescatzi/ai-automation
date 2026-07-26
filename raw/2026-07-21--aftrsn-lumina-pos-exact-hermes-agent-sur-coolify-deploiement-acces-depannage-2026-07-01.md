---
type: raw
title: "AFTRSN-LUMINA — POS EXACT — Hermes Agent sur Coolify (deploiement, acces, depannage) — 2026-07-01"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — pos exact — hermes agent sur coolify (deploiement, acces, depannage) — 2026-07-01.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS EXACT — Runbook : Hermes Agent sur Coolify (déploiement, accès, dépannage)

**Procédure Opérationnelle Standardisée (spécifique).**
**Cible :** Coolify (VPS Hetzner) · projet **AFTRSN-Automation / production** · **Hermes Agent (NousResearch)**.
**Date :** 2026-07-01.

---

## 1. Ce qui tourne

Compose à **2 services** :
- **hermes-agent** (`nousresearch/hermes-agent`, `gateway run`) — backend, clés API (OpenRouter/Anthropic/OpenAI/Google). **Pas de domaine public.**
- **hermes-webui** (`ghcr.io/nesquena/hermes-webui`) — UI web sur **port 8787** (`HERMES_WEBUI_HOST=0.0.0.0`). **Porte le domaine.**

**Accès :** **https://hermes.aftersunpeople.com** · gate mot de passe (var Coolify **`SERVICE_PASSWORD_HERMESWEBUI`**).

## 2. Rôle dans l'écosystème

Bras d'exécution : web (search/browse/extract/vision), génération d'images, TTS, `execute_code`, **skills** (mémoire procédurale), cron, sous-agents parallèles, **support MCP** (peut lire LUMINA-MCP). ⚠️ **Distinct** du « modèle Hermes local gratuit » : cet agent **appelle des API payantes**.

## 3. Exposer un service (règle d'or Coolify)

- Le domaine se met sur **le bon service et le bon port** via la variable **`SERVICE_FQDN_<SERVICE>_<PORT>`** (ex. `SERVICE_FQDN_HERMESWEBUI_8787=hermes.aftersunpeople.com`).
- `SERVICE_URL_*` = génère juste l'URL, **ne crée pas** la route. Utiliser **`SERVICE_FQDN_*`** pour la route Traefik.
- Domaine **avec `https://`** → déclenche Let's Encrypt (TLS). Domaine nu = http/cert auto-signé.
- **Un seul** service porte un domaine donné (sinon conflit Traefik → 404).
- Après édition compose/domaine : **Redeploy** complet (pas Restart).

## 4. Mot de passe WebUI

- Auto-généré : var **`SERVICE_PASSWORD_HERMESWEBUI`** (Configuration → Environment Variables, au niveau **ressource**, pas par service).
- Pour en définir un : éditer/ajouter cette var → Save → Redeploy. Choisir **fort** (la WebUI exécute des commandes / parcourt les fichiers).

## 5. Sécurité

- Accès via **Traefik 443** + mot de passe. Garder **80/443** ouverts.
- **Ne pas** exposer le **8787 en firewall public** (inutile avec Traefik ; risque d'accès direct sans TLS).

## 6. Dépannage (symptômes → cause)

| Symptôme | Cause probable | Fix |
|---|---|---|
| `404 page not found` | Traefik sans route : domaine sur mauvais service, ou `SERVICE_URL` au lieu de `SERVICE_FQDN`, ou domaine en double | mettre `SERVICE_FQDN_HERMESWEBUI_8787=<domaine>` ; retirer le domaine de hermes-agent ; Redeploy |
| `Not Secure` (http) | domaine sans schéma | mettre `https://` dans Domains |
| `Not Private` (cert auto-signé) | Let's Encrypt pas émis | domaine en `https://` + Redeploy + attendre 1–2 min ; vérifier email LE de l'instance |
| WebUI injoignable | bind localhost | `HERMES_WEBUI_HOST=0.0.0.0` |

## 7. Dette / à faire

- [ ] (Optionnel) servir un **modèle Hermes open-weight local** (endpoint OpenAI-compatible) pour le « workhorse gratuit » — nécessite GPU.
- [ ] Brancher Hermes (skills/web/fichiers) comme outil des agents (Ops).
- [ ] Durcir : rotation du mot de passe, MCP auth.

## 8. Leçons (mémo)

- `SERVICE_FQDN_*` crée la route ; `SERVICE_URL_*` non.
- `https://` requis pour le cert Let's Encrypt.
- Un domaine = un service.
- Deux « Hermes » : l'**agent** (exécution, payant) vs le **modèle local** (gratuit, GPU).
