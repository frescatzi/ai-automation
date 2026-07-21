---
type: raw
title: "POS-EXACT_Elimination-OpenRouter-LLM-direct_2026-07-15"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_elimination-openrouter-llm-direct_2026-07-15.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-EXACT — Élimination d'OpenRouter du stack LUMINA (→ OpenAI direct)

**Date :** 2026-07-15 · **Projet :** LUMINA OS · **Objet :** sortir OpenRouter de tout le stack (ne pas payer un intermédiaire en plus des abonnements directs + VPS) → **OpenAI en direct partout**. La clé OpenAI est déjà installée côté Hermès et en credential n8n.
**Statut :** ✅ **FAIT, VALIDÉ LIVE & CLEANUP EFFECTUÉ.** Les 4 fronts migrés + **clé OpenRouter retirée d'Hermès et credential n8n « OpenRouter account » supprimé** (Karter, 15-07). Métriques n8n après suppression : failure rate **0,3 %** (−6,5 pp), run time moyen **3,47 s** (−2,73 s) → aucune régression, plus rapide sans l'intermédiaire.

---

## 1. Découverte structurante (UI Hermès `Settings > Providers`)

Hermès expose un **catalogue OpenRouter (763 modèles)** ET des **clés directes** (OpenAI, Anthropic, Gemini, DeepSeek configurées). Le routage se fait par le **nom de modèle** :
- **`~provider/model`** (préfixe **tilde**) = **provider DIRECT** (clé native). Ex. `~openai/gpt-mini-latest` = OpenAI direct.
- **`provider/model`** (sans tilde) = **via OpenRouter**. Ex. `openai/gpt-5.2` = OpenRouter (d'où le 402 même en « choisissant OpenAI »).

Le provider OpenAI-direct n'expose que 4 modèles : `gpt-5.5`, `gpt-5.5-mini`, `gpt-5.4-mini`, `gpt-5.4`. **Leçon clé : le préfixe `~` = direct ; sans tilde = OpenRouter.**

---

## 2. Les 4 fronts traités

### 2.A Hermès (backend) — le principal
`Settings > Preferences > Default Model` basculé sur **`~openai/gpt-mini-latest` (OpenAI direct)**. Le node n8n `Hermes-Agent` crée une nouvelle session à chaque appel → prend ce Default Model. **Test exec #10286 : vraie réponse, `outcome:success`, plus aucun 402/OpenRouter.** (max_tokens 128k devient inoffensif en direct : OpenAI facture l'usage réel, pas de réservation de crédit.)

### 2.B Agents AFTRSN (n8n)
Node LLM `lmChatOpenRouter` → **`lmChatOpenAi` gpt-4o-mini** (cred **`OpenAi account`** `fqQvWAsn0tBEhSGU`), en gardant le nom du node (connexion `ai_languageModel` préservée) :
- **AFTRSN-Marketing** `wyUEojwZjSKFRaRL` (était `~anthropic/claude-sonnet-latest`) ✅
- **AFTRSN-CHANNEL-CONTENT** `MWUwLi3vu2Ws65lL` ✅
- Les 6 autres agents étaient déjà en `lmChatOpenAi`. → **8 agents en OpenAI direct.**

> **MàJ 15-07 (allocation modèle) :** Marketing + Channel-Content ont ensuite été **re-basculés sur Claude Sonnet 5 direct** (`lmChatAnthropic`, cred « Anthropic account ») car ce sont des rôles de **rédaction/voix de marque** — voir le POS dédié `POS-EXACT_Allocation-modeles-LLM_2026-07-15.md`. Ça reste **direct** (Anthropic, pas OpenRouter), donc cohérent avec l'objectif « zéro intermédiaire ».

### 2.C LUMINA-AI-Router `Cu8SozYmondKM8RB`
Routeur HTTP multi-LLM par `task_type`. HTTP Request `openrouter.ai/chat/completions` → **`api.openai.com/v1/chat/completions`** ; auth `predefinedCredentialType` **openRouterApi → openAiApi** (OpenAi account) ; **remap** dans le Code node :
- `~anthropic/claude-sonnet-latest` (copy/reasoning/analysis/orchestrate) → **`gpt-4o`**
- `~openai/gpt-mini-latest` (draft) → **`gpt-4o-mini`**
- `~openai/gpt-latest` (image_gen) → **`gpt-4o`**
- `~google/gemini-flash-latest` (research/summarize/translate/verify/classify/extract/preprocess/realtime/multimodal) → **`gpt-4o-mini`**

**Testé live** (Execute) : exécution OK, `model=gpt-4o-mini`, HTTP OpenAI 200. Devient **OpenAI-only** (perte du multi-provider, cohérent avec « OpenAI partout »).

### 2.D LUMINA-MEMORY-CONSOLIDATION/NIGHTLY `dVZyCwGYqnqMpS3P`
2 nodes LLM (`Consolidate (LLM)` + `Skills LLM`) : `openrouter.ai` → **`api.openai.com`**, cred openAiApi, modèle `~anthropic/claude-sonnet-latest` → **`gpt-4o-mini`**. Les nodes `Embed` utilisaient déjà OpenAI. Publié.
> ⚠️ La consolidation tournait sur **Claude Sonnet** (qualité). En `gpt-4o-mini` : si la qualité des insights baisse, bumper à `gpt-4o`.

---

## 3. Méthode (édition n8n via store Pinia)

Pour un **node LLM langchain** : muter `node.type` → `lmChatOpenAi`, `typeVersion` 1.3, `parameters.model={__rl:true,value:'gpt-4o-mini',mode:'list',cachedResultName:'gpt-4o-mini'}`, `node.credentials={openAiApi:{id,name}}`, **garder le nom** (connexion préservée). Pour un **node HTTP Request** : `parameters.url` → OpenAI, `parameters.nodeCredentialType` → `openAiApi`, `node.credentials={openAiApi:{…}}`, et remplacer les strings de modèle dans le(s) Code node(s). Puis **vrai drag** sur un node réel pour marquer dirty → **Publish**.

---

## 4. Cleanup — FAIT (15-07)

Pour **arrêter tout paiement OpenRouter** — **effectué par Karter** :
1. ✅ Hermès `Settings > Providers > OpenRouter` → **clé API retirée**.
2. ✅ n8n → credential **« OpenRouter account »** `w8bIvwIxqs1v3S0g` **supprimé**.
3. Résultat : aucune régression (failure rate en baisse, run time en baisse) → confirme que plus aucun chemin critique ne dépendait d'OpenRouter. Reste par prudence : au prochain passage du cron de consolidation (3h) et via le Sentinel, surveiller toute alerte `openrouter`/credential manquant (workflow résiduel non-MCP).

---

## 5. Difficultés / Solutions / Lessons learned

### Difficultés
- « Choisir un modèle OpenAI » ne suffisait pas : le 402 OpenRouter persistait car le modèle sélectionné (`OpenAI: GPT-5.2`) venait du **catalogue OpenRouter**.
- Pas de champ `base_url` dans Coolify : la config providers d'Hermès est **dans l'UI** (`Settings > Providers`), pas en env.
- Le Router/consolidation authentifiaient via credential prédéfini `openRouterApi`.

### Solutions
- Repérer le **préfixe `~`** = provider direct → `~openai/gpt-mini-latest`.
- Basculer le **Default Model** d'Hermès (UI) ; swap des nodes n8n via store (type/URL/credential + remap modèle).
- Réutiliser le credential prédéfini **`openAiApi`** (« OpenAi account ») pour les nodes HTTP → pas de secret manipulé en clair.

### Lessons learned (pièges figés)
- **`~model` = direct, `model` sans tilde = via OpenRouter** (convention Hermès/HermesWebUI).
- **Un « modèle OpenAI » peut être servi par OpenRouter** : vérifier le **provider/base_url réel**, pas juste le label.
- Swap LLM n8n via store en **gardant le nom du node** = connexion préservée ; auth HTTP via **credential prédéfini** (jamais la clé en clair).
- OpenAI direct = pas de réservation de crédit pré-vol (contrairement à OpenRouter) → le gros `max_tokens` cesse d'être bloquant.

---

*POS-EXACT — Élimination OpenRouter, 2026-07-15. OpenAI direct partout (Hermès + 8 agents + Router + consolidation), validé live. n8n fait foi ; le préfixe `~` distingue direct vs OpenRouter côté Hermès.*
