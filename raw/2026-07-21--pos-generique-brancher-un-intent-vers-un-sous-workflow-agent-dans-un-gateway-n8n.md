---
type: raw
title: "POS-GENERIQUE_Brancher-un-intent-vers-un-sous-workflow-agent-dans-un-gateway-n8n"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_brancher-un-intent-vers-un-sous-workflow-agent-dans-un-gateway-n8n.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Brancher un intent vers un sous-workflow agent dans un gateway n8n

**Scope :** réutilisable (toute marque, tout agent, tout gateway routé par intent).
**But :** quand un classifieur d'intention produit un nouvel intent, router ce cas vers un **sous-workflow agent** et renvoyer sa réponse, sans casser les branches existantes.

## Modèle
`Gateway (trigger → parse → auth → classifieur d'intention) → chaîne d'IF de routage → branches`. On ajoute un intent + une branche dédiée.

## Procédure

### 1. Déclarer le nouvel intent (classifieur)
1. Ouvrir le node classifieur (LLM chain).
2. **Inspecter la valeur LIVE** de l'enum d'intents (le séparateur réel peut différer de l'export : `|`, `\n`, virgule…).
3. Ajouter le nouvel intent par **modif ciblée** (remplacer une sous-chaîne existante), pas en retapant tout.
4. Ajouter une règle décrivant quand l'émettre. Fermer/rouvrir pour vérifier, puis publier.
5. **Vérifier les drapeaux annexes** (ex. `needs_clarification`, `requires_validation`) : s'assurer qu'ils n'entrent pas en conflit avec le routage du nouvel intent (une porte de garde peut se déclencher AVANT le routage).

### 2. Rendre l'agent cible appelable
1. Sur le workflow agent, ajouter un trigger **« When Executed by Another Workflow »** (mode « Accept all data ») et le relier à l'**entrée principale** de l'agent, en plus de son trigger habituel.
2. Régler la source du prompt de l'agent sur **« Define below »** avec une expression robuste : `{{ $json.<champ1> || $json.<champ2> || $json.<champ3> }}` (pour lire l'entrée quel que soit le déclencheur).
3. Publier. Noter le **champ de sortie** de l'agent (souvent `output`).

### 3. Ajouter la branche dans le gateway
1. **Réutiliser** un node d'envoi existant : copier/coller le node de réponse d'une branche voisine (garde la credential) ; renommer ; adapter les expressions (`chat_id` depuis le node de classification ; `text` depuis la sortie de l'agent, ex. `{{ $json.output }}`).
2. Ajouter un node **Execute Sub-workflow** ciblant l'agent (« Run once with all items ») ; le relier au node de réponse. L'item courant contient déjà les champs utiles (query, chat_id) → pas besoin de node « Prepare » si le trigger agent est en « Accept all data ».
3. Ajouter un node **If** testant `{{ $('<node classification>').first().json.intent }}` == `<nouvel_intent>`.
4. Insérer l'IF sur la branche cul-de-sac : **supprimer** l'ancien lien (source→dead-end), relier **source→If**, **If.true→[branche agent]**, **If.false→dead-end**.
5. **Vérifier la table des connexions via le DOM** avant de publier. Publier.

### 4. Vérification
- Exécuter l'agent isolément (trigger chat) pour confirmer le champ de sortie.
- Vérifier chaque lien attendu (DOM).
- Test bout-en-bout depuis le canal réel.

### 5. Rollback
- Retirer l'intent + la règle ; retirer le trigger execute + rétablir la source de prompt ; supprimer If/Execute/Reply et rétablir le lien source→dead-end. n8n conserve les versions publiées.

---

## Difficultés rencontrées (génériques)
- Session du navigateur d'automatisation perdue au moindre rechargement d'URL.
- Export d'un champ ≠ valeur live (séparateurs, encodage).
- Édition programmatique non committée si un overlay masque le champ.
- Câblage de connecteurs proches au pixel, source d'erreurs silencieuses.
- Une **porte de garde** (clarification/validation) placée avant le routage peut intercepter le nouvel intent.

## Solutions implémentées (génériques)
- Travailler **en clics internes SPA** (jamais de full navigate) après login dans l'onglet piloté.
- **Inspecter la valeur live** et éditer par **modif ciblée**.
- Setter natif + events + **fermer/rouvrir** pour vérifier.
- **Vérifier les connexions par le DOM** (`data-source-node-name`/`data-target-node-name`).
- Réutiliser par **copier-coller** un node crédentialé plutôt que reconfigurer.
- Auditer les **drapeaux de garde** du classifieur pour le nouvel intent.

## Lessons learned (génériques)
1. Automatiser n8n = **clics SPA uniquement**.
2. **Live > export** : toujours vérifier la valeur réelle avant d'éditer.
3. **Fermer/rouvrir** = seule preuve fiable qu'un champ a committé.
4. **DOM = source de vérité** pour les connexions.
5. Rendre un agent **double-déclenchable** = source de prompt « Define below » + expression multi-champs.
6. Un nouvel intent doit être **exempté des portes de garde** (clarification/validation) si l'agent cible gère lui-même ce contexte, sinon risque de boucle.
