---
type: raw
title: "POS-GENERIQUE_double-trigger-agent-et-memoire"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_double-trigger-agent-et-memoire.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Rendre un agent n8n appelable par un autre workflow SANS casser sa mémoire (+ router un intent auto-géré sans boucle de clarification)

**Type :** Marche-à-suivre GÉNÉRIQUE (abstraite, réutilisable pour n'importe quel workflow / marque)
**Portée :** tout agent LangChain n8n déclenché à la fois par un **Chat Trigger** et par un **Execute Workflow Trigger**,
et tout **classifieur d'intention** en amont dont une porte de clarification précède le routage.

---

## 1. Problème générique

Quand un agent conçu pour le **Chat Trigger** est aussi appelé par un **autre workflow** (`executeWorkflowTrigger`) :

1. La **source du prompt** de l'agent (« Connected Chat Trigger Node ») ne lit pas l'entrée de l'appel workflow.
2. La **mémoire de conversation** (Postgres/Redis Chat Memory) en mode « Connected Chat Trigger Node » attend un
   champ `sessionId` produit par le Chat Trigger — **absent** sur l'appel workflow → erreur *« No session ID found »*.
3. En amont, si un **routeur** met un flag `needs_clarification=true` et que la **porte de clarification est franchie
   AVANT le routage d'intent**, une intention qui gère elle-même son contexte (ex. un orchestrateur) part en
   **boucle de clarification sans état**.

---

## 2. Procédure (numérotée à partir de 1)

### Partie A — Rendre l'agent double-déclenchable

1. Ajouter un node **When Executed by Another Workflow** (`executeWorkflowTrigger`, `inputSource = passthrough` /
   « Accept all data ») et le brancher sur **l'entrée principale** de l'agent, en plus du Chat Trigger existant.
2. Passer la **source du prompt** de l'agent en **« Define below »** avec une expression tolérante aux deux chemins,
   ex. `{{ $json.chatInput || $json.query || $json.message }}`.

### Partie B — Adapter la mémoire au double-déclencheur (LE piège souvent oublié)

3. Ouvrir le node **Chat Memory** de l'agent.
4. **Session ID** : passer de « Connected Chat Trigger Node » à **« Define below »**.
5. Champ **Key** : basculer **Fixed → Expression** et saisir une clé robuste couvrant les deux triggers :
   `{{ $json.sessionId || $json.<clef_appel_workflow> || "<valeur_par_defaut>" }}`
   (ex. `<clef_appel_workflow>` = `chat_id`, `session_id`, `user_id`… selon ce que le workflow appelant transmet).
6. Vérifier la valeur exacte (1 seule paire `{{ }}`, pas d'espace parasite, littéral de secours entre guillemets).
7. Committer (cliquer hors du champ), fermer le node, **Publish**.

### Partie C — Empêcher la boucle de clarification pour un intent auto-géré

8. Dans le **classifieur d'intention**, ajouter une règle explicite : pour l'intent qui gère lui-même son contexte,
   **forcer `needs_clarification=false`** (et un défaut de contexte, ex. brand par défaut).
9. Ajouter une **exception** à la règle d'ambiguïté qui déclenche la clarification :
   `… si ambigu (SAUF intent="<intent_auto_gere>") → needs_clarification=true`.
10. Modifier **la valeur live** de façon **ciblée** (ne jamais retaper tout le prompt). **Publish**.

---

## 3. Vérification générique

- L'agent renvoie sa réponse sur les DEUX chemins (chat direct + appel workflow) sans erreur mémoire.
- Le contrat de sortie (champ `output` ou équivalent) est non vide sur l'appel workflow.
- Une requête relevant de l'intent auto-géré n'entraîne plus de question de clarification en boucle.
- Le workflow appelant reçoit la réponse et son node d'envoi renvoie un succès (`ok:true` pour Telegram).

## 4. Rollback générique

- Restaurer la version antérieure du/des workflow(s) via l'historique des versions, ou annuler chaque modif ciblée
  (source de prompt, Session ID/Key, règles du classifieur) puis Publish.

---

## 5. Difficultés rencontrées (génériques)

- L'adaptation « source du prompt » est **visible** mais l'adaptation **mémoire** est silencieuse et oubliée →
  l'erreur n'apparaît que lors du 1er appel réel par le workflow.
- Les **dropdowns** (el-select) et **éditeurs d'expression** des NDV n8n résistent parfois aux clics/frappes simulés.
- Les **portes conditionnelles** placées avant le routage d'intent créent des effets de bord non intuitifs.

## 6. Solutions implémentées (génériques)

- Traiter le double-déclencheur comme un **triptyque** : prompt + mémoire + (le cas échéant) règles du routeur amont.
- Clé de session **défensive** avec `||` et **valeur de secours** littérale, pour ne jamais retomber sur `undefined`.
- Pour un intent auto-géré, neutraliser la clarification **au niveau du classifieur** (le plus simple, sans état).
- Éditions **ciblées** de valeurs live + **vérification** systématique après coup.

## 7. Lessons learned (pièges figés, réutilisables)

- **« Rendre un agent appelable » = 2 chantiers, pas 1** : entrée (prompt) **ET** mémoire (session key).
  Oublier la mémoire = erreur *« No session ID found »* garantie sur le path appel-workflow.
- **Clé de session universelle** : `{{ $json.sessionId || $json.<clef> || "<défaut>" }}` couvre chat + appel workflow.
- **Clarification sans état = boucle** : si la réponse de clarification est reclassée comme un nouveau message isolé,
  elle ne résout jamais. Tant qu'il n'y a pas d'état de conversation, **éviter de déclencher la clarification** pour
  les intents qui n'en ont pas besoin.
- **Ordre des portes** : vérifier qu'une porte de clarification ne court-circuite pas le routage d'intent.
- **Orchestrateur = coût** : un agent qui appelle tous ses sous-agents est lent/coûteux ; prévoir un garde-fou
  (sélection des spécialistes pertinents, budget d'appels).

---
*Convention : numérotation à partir de 1. Double sauvegarde : projet Claude + Google Drive « LUMINA AI DOCS ».*
