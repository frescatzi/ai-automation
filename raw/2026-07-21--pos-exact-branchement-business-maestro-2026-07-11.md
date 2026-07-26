---
type: raw
title: "POS-EXACT_Branchement-Business-Maestro_2026-07-11"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_branchement-business-maestro_2026-07-11.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS EXACT — Branchement Business → Maestro (Telegram / Lyra)

**Date :** 2026-07-11 · **Scope :** LUMINA/AFTRSN · **Boussoles :** « Spec Build Lumina Telegram » + « Lumina Bible 360 »

**But :** router les questions business/décision reçues sur Telegram vers l'agent orchestrateur **AFTRSN-Maestro**, et renvoyer sa réponse à l'utilisateur.

**Préconditions :** être connecté à n8n **dans l'onglet exact piloté** (email+mdp+2FA) ; ensuite **navigation par clics internes uniquement** (jamais de rechargement d'URL, sinon perte de session — voir Lessons).

---

## Étape 1 — Router : ajouter l'intent `maestro`
Workflow **`LUMINA-TELEGRAM-INTENT-ROUTER`** (`KTLwQi7ZKHDTexMZ`).
1. Ouvrir le node **« Classifieur d'intention »**.
2. Dans le System Message (champ « Message »), **modif ciblée** : remplacer `hermes_ops|validation_response` par `hermes_ops|maestro|validation_response` (l'enum est séparé par des **`|`**, pas par des retours ligne).
3. Ajouter la règle : *intent="maestro" pour toute question de DÉCISION/STRATÉGIE/BUSINESS (prix, positionnement/décision de marque, nouveau produit/projet, amélioration de process, "devrait-on…", "est-ce une bonne idée de…"). C'est de l'orchestration, pas une simple recherche.*
4. Fermer le node, **rouvrir** pour vérifier la persistance, puis **Publish** (renseigner un nom de version + description).

## Étape 2 — Maestro : le rendre appelable
Workflow **`AFTRSN-Maestro`** (`OlrQO21u178SjgBK`).
1. Node agent **« AFTRSN-Maestro »** → « Source for Prompt (User Message) » = **« Define below »** ; Prompt = `{{ $json.chatInput || $json.query || $json.message }}`.
2. Ajouter un node trigger **« When Executed by Another Workflow »** (Execute Sub-workflow → Triggers) → « Input data mode » = **« Accept all data »**.
3. Relier la **sortie** de ce trigger à l'**entrée principale** de l'agent (drag ; vérifier la connexion via le DOM).
4. **Publish**.
- Contrat confirmé (test MCP, exec 3826) : Maestro renvoie dans le champ **`output`**.

## Étape 3 — Gateway : brancher intent==maestro
Workflow **`LUMINA-TELEGRAM-GATEWAY`** (`uGF3pSv6aTK79cqi`).
1. Copier le node Telegram **« Reply to user (web) »** (Ctrl+C/Ctrl+V) → renommer **« Reply (maestro) »** (garde la credential `LUMINA-Lyra - Telegram`). Régler : `Chat ID = {{ $('Classify the intent').first().json.chat_id }}` ; `Text = {{ $json.output }}`.
2. Ajouter un node **Execute Sub-workflow → « Execute A Sub Workflow »** → cible **AFTRSN-Maestro** (mode « Run once with all items »). Il devient **« Call 'AFTRSN-Maestro' »**. Relier sa sortie → **« Reply (maestro) »**.
3. Ajouter un node **If** → renommer **« Is it Maestro? »** ; condition (Expression) `{{ $('Classify the intent').first().json.intent }}` **is equal to** `maestro`.
4. Relier **If.true → Call 'AFTRSN-Maestro'** et **If.false → Reply (module pending)**.
5. **Supprimer** l'ancien lien `Is it a web/ops question?` (FALSE) → `Reply (module pending)` (survol de la connexion → icône corbeille), puis relier **`Is it a web/ops question?` (FALSE) → Is it Maestro?**.
6. Vérifier la table des connexions via le DOM, puis **Publish**.

## Vérification
- Exécuter Maestro via MCP (chatInput = question business) → confirme `output` non vide.
- Vérifier via DOM les 6 liens : web/ops→Send typing ; web/ops→Is it Maestro? ; Is it Maestro?→Call 'AFTRSN-Maestro' ; Is it Maestro?→Reply (module pending) ; Call→Reply (maestro).
- Test réel Telegram (question business).

## Rollback
- Router : retirer `maestro` de l'enum + la règle. Maestro : retirer le trigger execute + remettre la source « Connected Chat Trigger Node ». Gateway : supprimer If/Call/Reply(maestro) et rétablir web/ops(FALSE)→Reply (module pending). (n8n garde les versions publiées → restaurer une version antérieure.)

---

## Difficultés rencontrées
- **Perte de session du navigateur piloté** à chaque rechargement d'URL (401, autosave « Unauthorized »).
- **Enum séparé par `|`** (et non `\n` comme le laissait croire l'export JSON).
- **Champ « Text » qui ne committait pas** quand un popup de résultat masquait le champ.
- **Câblage true/false d'un IF au pixel** (dots espacés de ~17 px) fragile.
- **Agent non appelable** via execute trigger tant que la source du prompt reste « Connected Chat Trigger Node ».

## Solutions implémentées
- Se connecter dans l'onglet EXACT piloté, puis **clics SPA uniquement** (Overview → recherche → clic workflow), zéro rechargement.
- **Modif ciblée** de la valeur live (remplacement de sous-chaîne), jamais retaper tout le prompt.
- Setter natif + events `input/change`, puis **fermer/rouvrir le node** pour vérifier la persistance.
- Vérifier/poser les connexions via le **DOM** (`data-source-node-name` / `data-target-node-name`) ; suppression via survol → corbeille.
- Source « Define below » + `{{ $json.chatInput || $json.query || $json.message }}` + execute trigger sur l'entrée principale.

## Lessons learned
1. **n8n = clics SPA only** dans le navigateur piloté ; un full navigate tue la session.
2. Toujours **inspecter la valeur LIVE** d'un champ avant de l'éditer.
3. **Vérifier après coup** (fermer/rouvrir) toute édition programmatique de champ.
4. **Vérifier la table des connexions** (DOM) avant de publier.
5. Un AI Agent doit lire l'entrée **quel que soit le trigger** (source « Define below »).
6. Sortie d'un AI Agent appelé via Execute Workflow = champ **`output`**.

> **⚠️ Bug ouvert (prochain point) :** le classifieur met `intent=maestro` **et** `needs_clarification=true`, donc la porte « Need clarification? » (avant le routage) fait boucler la clarification de marque. Fix = forcer `needs_clarification=false` pour `maestro` dans le Router. Voir la passation du 11-07.
