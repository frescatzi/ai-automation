---
type: raw
title: "POS-EXACT_Hermes-fiabilite-garde-fou-erreurs-backend_2026-07-15"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_hermes-fiabilite-garde-fou-erreurs-backend_2026-07-15.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-EXACT — Hermès · Fiabilité latence : diagnostic + garde-fou erreurs backend

**Date :** 2026-07-15 · **Projet :** LUMINA OS · exécuteur **Hermès** · **Workflow :** `LUMINA-Hermes-Agent` (`Dwv4rcMqNAyQzlrF`), node **`Hermes-Agent`** (Code)
**Objet :** fiabiliser le comportement d'Hermès. **Constat central : ce n'est plus un problème de latence** — la tuyauterie est déjà rapide. Le blocage actuel est **côté backend** (crédits LLM). Ajout d'un **garde-fou n8n** pour que les erreurs backend soient traitées proprement.
**Statut :** ✅ **FAIT, PUBLIÉ & VALIDÉ LIVE** (exec #9928 : `outcome='fail'` + message propre).

---

## 1. État live constaté (2026-07-15)

Le fix « approbation + latence » est **déjà en service** dans le node `Hermes-Agent` (MAJ 11-07) :
- **Auto-approbation** HTTP `POST /api/approval/respond {choice:'once', approval_id:null}` à chaque tour (débloque les outils gated tirith).
- **Détection de fin fiable** : `active_stream_id===null && !pending_user_message && message assistant présent`.
- **Hermès-first** : plus de pré-mâchage (embeddings/skills pgvector/CANON-INJECT retirés) ; prompt SYS « utilise toujours tes outils + format Telegram ».
- **`POST /api/reasoning {effort:'low'}`** (vitesse GUI).
- Poll **400 ms**, borne **50 s** wall-clock (sous la limite runner 60 s).

**Mesures live (MCP `execute_workflow`) :** round-trip total **~3–6 s** (login ~0,5 s + Hermes-Agent ~2,5–3 s). **La latence n'est pas le problème.**

## 2. Diagnostic — le vrai blocage (reproductible)

Sur **toute** requête (même « dis bonjour »), Hermès renvoie en ~2,5 s :

> **Out of credits: Error code 402** — *This request requires more credits, or fewer max_tokens. You requested up to **128000** tokens, but can only afford **87882**.* (openrouter.ai/settings/credits)

Backend = `HermesWebUI/0.51.92` (header serveur). Donc :
- **(a)** le compte **OpenRouter du backend est à court de crédits** ;
- **(b)** chaque requête demande **`max_tokens` = 128 000** (énorme → cher **et** lent). Le message dit littéralement « *or fewer max_tokens* ».

**Défaut n8n dérivé (corrigé ici) :** le node renvoyait cette erreur avec **`outcome:'success'`** → (1) `Build Log` l'aurait comptée comme un **skill réussi** (corruption de `skill_registry` : `successes++`, promotion `maturity='autonomous'`), et (2) le **JSON brut** d'OpenRouter serait livré tel quel dans Telegram.

## 3. Actions requises CÔTÉ BACKEND (Karter — hors n8n)

Tant que ce n'est pas fait, Hermès ne peut **pas** répondre (il renverra le message d'indisponibilité) :
1. **Recharger les crédits OpenRouter** du backend Hermès ; **et/ou**
2. **Baisser le `max_tokens`** du modèle (128k → p.ex. 8k–16k) via `hermes model` / config backend — **meilleur levier** : réduit coût ET latence d'un coup ; **et/ou**
3. **Changer de fournisseur** LLM via `hermes model`.

## 4. Correctif n8n appliqué — garde-fou erreurs backend

Dans `Hermes-Agent`, juste avant le `return`, insertion d'un bloc (édité in-page par `jsCode.replace(anchor,…)`, sans exfiltrer le code) :

```js
let __d='';
{ const low=String(text||'').toLowerCase();
  if(outcome==='success' && /out of credits|error code:|402|payment required|insufficient|quota|rate limit|429|hermes erreur|pas de reponse/.test(low)){
    __d=String(text).slice(0,200);
    outcome='fail';
    text='Hermes indisponible pour le moment (quota ou credits du backend LLM). Reessaie plus tard.';
  } }
return [{json:{text, outcome, usedTools, detail:__d}}];
```

- **`outcome='fail'`** sur erreur détectée → `Build Log` route vers la branche échec (`failures++`, `maturity='supervised'`) : **skill_registry non corrompu**.
- **Message propre** livré à l'utilisateur au lieu du JSON brut.
- **`detail`** = erreur brute tronquée (200 c.) pour l'observabilité n8n (non transmise à l'utilisateur ; `Result` ne forwarde que `text/outcome/usedSkills`).
- Le timeout (`outcome='timeout'`) était déjà hors de la branche succès — inchangé.

## 5. Validation live

| Test | Avant | Après (exec #9928) |
|---|---|---|
| « dis bonjour » (backend sans crédits) | `outcome:'success'` + JSON 402 brut | **`outcome:'fail'`** + « Hermes indisponible… » ; `detail` = 402 tronqué |
| `Build Log` | branche succès (corruption) | **branche échec** (correct) |
| Latence tuyauterie | ~3 s | ~3 s (inchangée) |

## 6. Difficultés / Solutions / Lessons learned

### Difficultés rencontrées
- Le symptôme historique « pas de réponse / latence » masquait deux causes **successives** : d'abord le blocage d'approbation (déjà résolu), **puis** l'épuisement des crédits backend — invisible car renvoyé en `outcome:'success'`.

### Solutions implémentées
- **Mesurer d'abord** (exécution MCP directe ×2) au lieu de supposer la latence → révèle instantanément le 402.
- Garde-fou déterministe : toute réponse d'erreur backend → `outcome='fail'` + message propre + `detail` observable.

### Lessons learned (pièges figés)
- **« Latence » ≠ latence** : toujours **mesurer** avant d'optimiser. Ici la tuyauterie était déjà à ~3 s ; le vrai blocage était les **crédits/`max_tokens` du backend**.
- **Un exécuteur ne doit jamais renvoyer `success` sur un texte d'erreur** : sinon il pollue la gouvernance des skills (maturité) et livre du bruit à l'utilisateur. Détecter les patterns d'erreur et **downgrader en `fail`**.
- **`max_tokens` surdimensionné = coût + latence** : le plafonner est un double gain.
- Édition n8n : `jsCode.replace(anchor,…)` in-page (jamais exfiltrer le code) + **vrai drag** pour marquer dirty avant Publish.

---

*POS-EXACT Hermès fiabilité — 2026-07-15. Le fix n8n rend les échecs propres ; **Hermès ne répondra réellement qu'après recharge des crédits / baisse du max_tokens côté backend**. n8n fait foi.*
