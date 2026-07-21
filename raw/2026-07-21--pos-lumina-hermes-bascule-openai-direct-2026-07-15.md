---
type: raw
title: "POS-LUMINA_Hermes-bascule-OpenAI-direct_2026-07-15"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_hermes-bascule-openai-direct_2026-07-15.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# 🛠️ MARCHE À SUIVRE — Basculer Hermès d'OpenRouter → OpenAI direct + plafonner max_tokens

**Date :** 2026-07-15 · **Projet :** LUMINA OS · backend **Hermès** (`hermes.aftersunpeople.com`, `HermesWebUI/0.51.92`, hébergé Coolify)
**Objectif :** sortir d'OpenRouter (trop cher / marge de routage) au profit d'**OpenAI en direct**, et **plafonner `max_tokens`** (128k → ~4–8k) pour couper la sur-conso de tokens. **La clé OpenAI est déjà installée sur Hermès.**
**Exécution :** côté serveur/Coolify (mécanisme B). Claude valide en live après coup.

---

## 0. Le piège à éviter (à lire avant tout)

**« Modèle OpenAI » ≠ « OpenAI direct ».** OpenRouter sert aussi des modèles `openai/gpt-4o-mini`. Si le backend garde `base_url = https://openrouter.ai/api/v1`, changer le nom du modèle **reste sur OpenRouter**. Pour vraiment sortir d'OpenRouter, il faut que le **provider / base_url** d'Hermès soit **`https://api.openai.com/v1`** avec la **clé OpenAI** (pas la clé OpenRouter). C'est LE geste qui compte.

---

## 1. Repérer la surface de config (2 endroits possibles)

Hermès se configure soit par **CLI `hermes model`**, soit par **variables d'env** (Coolify). Vérifier les deux :

- **CLI** (dans le conteneur Hermès / shell serveur) :
  ```
  hermes model            # liste les modèles + le courant
  hermes model --help     # syntaxe exacte de bascule (à confirmer, propre à HermesWebUI)
  hermes --help           # voir s'il y a `hermes config` / `hermes provider`
  ```
- **Env vars Coolify** (service Hermès) : chercher les clés du type
  `OPENAI_API_KEY`, `OPENAI_BASE_URL` / `LLM_BASE_URL` / `OPENROUTER_*`, `LLM_MODEL` / `MODEL`, `MAX_TOKENS` / `LLM_MAX_TOKENS`.

> Noter la valeur **actuelle** du base_url / provider (probablement `openrouter.ai`) → c'est ce qu'on remplace.

---

## 2. Basculer le provider sur OpenAI direct

Selon la surface trouvée :

- **Via env (Coolify)** — le plus fiable :
  - `LLM_BASE_URL` (ou équivalent) → **`https://api.openai.com/v1`**
  - la clé utilisée → **ta clé OpenAI** (déjà installée ; s'assurer que c'est bien elle qui est référencée, pas la clé OpenRouter)
  - `MODEL` / `LLM_MODEL` → **`gpt-4o-mini`** (recommandé : bon tool-calling, rapide, peu cher) — ou `gpt-5-mini` pour rester cohérent avec Maestro.
- **Via CLI `hermes model`** : sélectionner le modèle OpenAI **et vérifier** (étape 5) que la requête part bien vers `api.openai.com` et non `openrouter.ai`. Si `hermes model` ne change que le nom du modèle sans changer le provider, repasser par les env vars.

> **Choix du modèle :** Hermès est un **exécuteur qui utilise des outils** (terminal, web). `gpt-4o-mini` = cible idéale (fiable en tool-calling, ~30× moins cher qu'un gros modèle). Éviter un modèle « raisonneur » à effort élevé (brûle des tokens) — le node force déjà `reasoning:low`.

---

## 3. Plafonner max_tokens (le gros gain conso)

- Fixer le **max_tokens de sortie** à **4096** (suffisant pour des réponses Telegram ; monter à 8192 si des réponses sont tronquées).
- Où : même surface qu'au §1 — env `MAX_TOKENS` / `LLM_MAX_TOKENS`, ou un réglage `hermes config`/`hermes model`.
- Effet : la requête cesse d'exiger 128 000 tokens de réserve → passe le contrôle de crédit **et** coûte/latence bien moins.

---

## 4. Redéployer / redémarrer

- Si config par **env Coolify** : **Redeploy** du service Hermès (les env ne prennent qu'au redéploiement).
- Si **CLI** : suivre ce qu'indique `hermes model` (souvent effet immédiat ou après restart du service).

---

## 5. Vérifier (Claude, en live)

Dis-moi quand c'est redéployé. Je relance `execute_workflow` sur `Dwv4rcMqNAyQzlrF` avec une requête bénigne et je contrôle :
1. **Plus d'erreur 402 / « Out of credits »** ; Hermès renvoie une **vraie réponse**.
2. `outcome='success'` (le garde-fou ne se déclenche plus).
3. **Latence** end-to-end (cible : ~15–25 s pour une question web, quelques s pour du texte simple).
4. Contrôle du provider réellement appelé (via un test qui révèle la source, ou l'export session) = **OpenAI, pas OpenRouter**.

> Si après bascule ça repart en 402 mais côté OpenAI (« insufficient_quota »), c'est que le **compte OpenAI** est à sec → recharger côté OpenAI. Mais avec le crédit OpenAI déjà utilisé par Maestro, peu probable.

---

## 6. Rappels

- **max_tokens = réserve de sortie**, pas la conso réelle : 4–8k suffit largement pour Telegram ; c'est le levier n°1 contre la sur-conso.
- Le **garde-fou n8n** (POS 2026-07-15) reste en place : si un jour le backend re-échoue (crédits/quota), l'utilisateur voit un message propre, pas de JSON brut, et `skill_registry` n'est pas corrompu.
- Après validation, suite possible = **Phase B** (gouvernance des approbations : auto-approuver le non-critique, escalader vers Karter le critique).

---

*Marche à suivre — bascule Hermès OpenAI direct + max_tokens. 2026-07-15. Le geste clé = provider/base_url sur api.openai.com (pas juste le nom du modèle). n8n fait foi ; Claude valide en live après redéploiement.*
