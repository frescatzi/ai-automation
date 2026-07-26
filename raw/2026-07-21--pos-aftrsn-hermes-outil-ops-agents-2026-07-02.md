---
type: raw
title: "POS-AFTRSN_Hermes-outil-Ops-agents_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_hermes-outil-ops-agents_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# AFTRSN-LUMINA — POS EXACT — Brancher Hermes comme outil Ops des agents (A3) — 2026-07-02

État final : le Secrétaire et le Marketing peuvent déléguer une tâche opérationnelle à Hermes (web, fichiers, exécution) via un sous-workflow dédié, sans jamais exposer de secret.

## 0. Références

| Élément | Valeur |
|---|---|
| Sous-workflow | `LUMINA-Hermes-Exec` id `Dwv4rcMqNAyQzlrF` |
| Credential | Custom Auth `Hermes WebUI` id `LoNa2cG7DqHzzUHv` (domaine restreint `hermes.aftersunpeople.com`) |
| Agents câblés | Secrétaire `FtPFOKvi2t18DFb1`, Marketing `wyUEojwZjSKFRaRL` |
| API Hermes | `POST /api/auth/login {password}` (cookie `hermes_session`) → `POST /api/session/new` → `POST /api/chat/start {session_id,message}` → `GET /api/session/export?session_id=` |

## 1. Sécurité (décision)

- **Refusé** : `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` — exposerait TOUTES les variables d'env (clés, mots de passe DB) à n'importe quel node Code.
- **Retenu** : mot de passe Hermes dans un **credential Custom Auth** (isolé, jamais lisible par Code/expressions), avec **Allowed Domains** = `hermes.aftersunpeople.com` (le credential ne peut être transmis nulle part ailleurs).
- URL Hermes = publique, en dur dans le workflow (pas un secret).

## 2. Le sous-workflow LUMINA-Hermes-Exec

Triggers : chat (test) + `When Executed by Another Workflow` (jsonExample `{message}`) → **Hermes Login** → **Hermes Exec**.

**Node Hermes Login** (HTTP Request) : POST `https://hermes.aftersunpeople.com/api/auth/login` ; Authentication = Generic → **Custom Auth** (`Hermes WebUI`) ; **corps en mode champs nom/valeur vide** (`specifyBody:'keypair'`, `bodyParameters:{parameters:[]}`) pour que le `body:{password}` du credential s'injecte ; `contentType:'json'` ; options `fullResponse:true` (pour lire l'en-tête `set-cookie`) + `neverError:true`.

**Node Hermes Exec** (Code) : extrait le cookie (`set-cookie` → `hermes_session=…`) ; `session/new` ; `chat/start {session_id, message}` ; puis **poll** `GET /api/session/export` (paramètre via `qs`, `try/catch` dans la boucle pour absorber un 400 transitoire) toutes les 3 s jusqu'à ce que la réponse assistant soit **stable** (deux lectures identiques) ou 40 tours ; retourne `{text, session_id, model}`.

## 3. Brancher sur un agent

Ajouter un node `toolWorkflow` nommé `Call 'Hermes (Ops)'` → workflow `LUMINA-Hermes-Exec`, entrée `message = {{ $fromAI('message', '...') }}`, `description` = quand/comment utiliser Hermes (rappel : le Secrétaire ne fait que préparer, pas envoyer). Connexion `ai_tool` vers le node AI Agent. Publier.

## 4. Test (fiable)

Via API (le chat de l'éditeur est inutilisable en automation) :
```
POST /rest/workflows/:id/run
{ workflowData: {...w, pinData:{'When Executed by Another Workflow':[{json:{message:'...'}}]}}, runData:{}, triggerToStartFrom:{ name:'When Executed by Another Workflow' } }
```
Résultats validés le 02-07 : Hermes-Exec seul → 391 (17×23) ; Secrétaire → outil Hermes → 132 (44×3).

## 5. Pièges figés

1. **Custom Auth `body` ne fusionne pas en mode JSON brut** (`jsonBody`) → utiliser le mode champs nom/valeur (corps objet).
2. **n8n en mode queue** (n8n + n8n-worker) : le Code tourne sur le worker ; `$env` bloqué par défaut → ne pas dépendre de l'env, utiliser un credential.
3. **Chat de test fragile** → déclencher par API avec `triggerToStartFrom`.
4. `this.helpers.httpRequest` + `setTimeout` OK dans un node Code ; export via `qs` + try/catch (400 transitoire pendant l'initialisation du tour).
5. Prérequis : backend Hermes authentifié (voir POS Hermes/Coolify) — sinon réponses « Missing Authentication header ».

---
*POS EXACT — 2026-07-02, Claude (Cowork). Voir : POS-GENERIQUE agent→service authentifié par cookie, POS Hermes/Coolify, POS câblage Maestro.*
