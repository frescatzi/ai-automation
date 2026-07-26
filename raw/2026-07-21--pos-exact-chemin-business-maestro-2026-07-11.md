---
type: raw
title: "POS-EXACT_chemin-business-Maestro_2026-07-11"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_chemin-business-maestro_2026-07-11.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Faire aboutir les questions business Telegram → AFTRSN-Maestro

**Type :** Marche-à-suivre EXACTE (reproductible à l'identique sur CE système)
**Date :** 2026-07-11
**Projet :** LUMINA OS / AFTER SUN PEOPLE — Bot Telegram « Lyra »
**Objet :** Débloquer le chemin business (question de décision) depuis Telegram jusqu'à l'agent AFTRSN-Maestro, avec réponse réelle renvoyée dans Telegram.
**Résultat :** ✅ Validé bout-en-bout (Telegram delivered, `ok:true`, `message_id 92`).

---

## 0. Contexte / point de départ

Suite passation `PASSATION_LUMINA-business-Maestro_2026-07-11`, §3 : la question business
« Devrait-on lancer une gamme de produits solaires cet été ? » tournait en **boucle de clarification de marque**
(Lyra redemandait « LUMINA ou AFTRSN ? » sans jamais router vers Maestro).

Cause racine confirmée : le classifieur mettait `intent=maestro` **ET** `needs_clarification=true` (marque « none »),
or la porte `Need clarification?` du Gateway est franchie AVANT le routage d'intent → la question partait sur
« Ask which brand » et n'atteignait jamais la branche Maestro.

Deux correctifs ont finalement été nécessaires : (A) le Router, puis (B) la Chat Memory de Maestro
(révélée seulement une fois le routage réparé).

---

## 1. Préconditions

1. Chrome ouvert, piloté par l'agent ; cookies de session n8n partagés (login automatique).
2. Instance : `https://n8n.aftersunpeople.com`.
3. Accès en écriture aux workflows (scopes `workflow:update` + `workflow:publish`).

---

## 2. Étapes exactes réalisées (numérotées à partir de 1)

### Correctif A — Router `LUMINA-TELEGRAM-INTENT-ROUTER` (`KTLwQi7ZKHDTexMZ`)

1. Ouvrir n8n via l'onglet piloté : `https://n8n.aftersunpeople.com/home/workflows` (déjà loggé via cookies partagés).
2. Dans la barre « Search » de l'onglet Workflows, taper `INTENT-ROUTER` → cliquer sur la carte
   **LUMINA-TELEGRAM-INTENT-ROUTER** (navigation SPA, URL devient `/workflow/KTLwQi7ZKHDTexMZ`).
3. Double-cliquer le node **« Classifieur d'intention »**.
4. Dérouler jusqu'à **Chat Messages → System** → cliquer le champ **Message** → cliquer l'icône « ouvrir l'éditeur »
   (panneau agrandi à droite).
5. Appliquer **2 modifications ciblées** dans le prompt système (valeur live, jamais retaper tout) :
   - **Règle maestro** — après « …C'est de l'orchestration, pas une simple recherche. » ajouter :
     `Pour intent="maestro", mets TOUJOURS le flag de clarification a false et brand="aftrsn" par defaut (Maestro gere lui-meme le contexte de marque).`
   - **Règle marque ambiguë** — remplacer `si ambiguë ET utile →` par
     `si ambiguë ET utile (sauf intent="maestro") →`.
   - Passage prompt : **1418 → 1588 caractères**.
6. Fermer le node (Échap puis X du NDV) → le bouton **Publish** passe en jaune (workflow « dirty » = modif captée par le modèle).
7. Cliquer **Publish** → renseigner « Describe changes » → **Publish**. Version publiée **`f7fe2432`**.

### Test #1 (Telegram) → révèle un 2e blocage

8. Karter envoie en Telegram : « Devrait-on lancer une gamme de produits solaires cet été ? ».
9. Résultat : plus de boucle de marque (✅ routage OK), MAIS SENTINEL renvoie
   `Erreur : Error in sub-node Chat Memory` sur `AFTRSN-Maestro` (exécution **3888**, erreur en 64 ms).

### Correctif B — Chat Memory de `AFTRSN-Maestro` (`OlrQO21u178SjgBK`)

10. Ouvrir l'exécution en erreur : `/workflow/OlrQO21u178SjgBK/executions/3888` → « Open errored node ».
11. Diagnostic lu dans le node **Chat Memory** :
    - Session ID = **« Connected Chat Trigger Node »** → cherche un champ `sessionId` dans l'input.
    - `Session Key From Previous Node = {{ $json.sessionId }}` → **undefined** sur l'appel workflow.
    - Message : *« No session ID found. Expected to find the session ID in an input field called 'sessionId' ».*
    - Explication : sur le trigger **When Executed by Another Workflow** (appel Gateway), l'item passé contient
      `chat_id`/`query` mais **pas** `sessionId` → erreur immédiate. Le path Chat-Trigger, lui, marchait
      (d'où les exécutions antérieures en succès).
12. Onglet **Editor** → double-cliquer le node **Memory** (Chat Memory / Postgres).
13. **Session ID** : ouvrir le menu déroulant → sélectionner **« Define below »**.
14. Champ **Key** apparu → basculer le sélecteur **Fixed → Expression**.
15. Saisir l'expression (taper `{{` laisse l'éditeur fermer automatiquement, puis saisir l'intérieur seulement) :
    `{{ $json.sessionId || $json.chat_id || "aftrsn" }}`
    (vérifié : 1 paire d'accolades, `"aftrsn"` intact, aucun espace parasite).
16. Cliquer hors du champ (Table Name) pour committer → fermer le node → **Publish** (jaune) → **Publish**.
    Version publiée **`ca75c537`**.

### Test #2 (Telegram) → validation bout-en-bout

17. Karter renvoie la même question à Lyra.
18. `AFTRSN-Maestro` exécution **3907** : **Succeeded in 8m 44 s**, **~241 944 tokens** (Maestro appelle
    l'ensemble de ses sous-agents), Chat Memory ✅, champ **`output`** = recommandation structurée
    (« # Recommandation : ❌ Non — pas de gamme "sun care" classique cet été… »).
19. `LUMINA-TELEGRAM-GATEWAY` exécution **3905** : **Succeeded in 8m 50 s** ; node **« Reply (maestro) »**
    (Telegram sendMessage) = `Success 178ms`, sortie `ok: true`, `message_id: 92` → **message délivré dans Telegram**.

---

## 3. Vérification

- Router : prompt système = 1588 caractères, contient les 2 ajouts, publié `f7fe2432`.
- Maestro : Chat Memory en « Define below » + clé expression, publié `ca75c537`.
- Bout-en-bout : `output` Maestro non vide + Gateway `Reply (maestro)` `ok:true` `message_id:92`.
- Plus aucune boucle de clarification de marque sur une question business.

## 4. Rollback

1. Router : rouvrir `KTLwQi7ZKHDTexMZ`, restaurer la version antérieure à `f7fe2432` (Historique des versions),
   ou retirer les 2 phrases ajoutées → Publish.
2. Maestro : rouvrir `OlrQO21u178SjgBK`, remettre **Session ID = « Connected Chat Trigger Node »** (clé `{{ $json.sessionId }}`),
   ou restaurer la version antérieure à `ca75c537` → Publish.

---

## 5. Difficultés rencontrées

- **Router non éditable via le connecteur n8n MCP** : il n'apparaît pas dans le scope du connecteur (qui ne voit
  que les agents AFTRSN + Maestro) et le connecteur est en lecture/exécution seule (pas d'outil `update`).
  → obligation de passer par le navigateur piloté.
- **Dropdown « Session ID » insensible** aux clics simulés et au clavier (composant type el-select en overlay
  « teleporté »).
- **Autocomplétion de l'éditeur d'expression** : taper `{{ … }}` en entier ajoutait une paire `}}` en trop
  (`… }} }}` → clé invalide).
- **Fenêtre du navigateur qui se redimensionne** entre deux captures → coordonnées de clic qui se décalent.

## 6. Solutions implémentées

- Édition **100 % navigateur en clics SPA** (aucun rechargement d'URL pendant l'édition), login via cookies partagés.
- Modifications **ciblées** de la valeur live du prompt (jamais retaper l'ensemble), vérifiées par relecture DOM.
- Dropdown récalcitrant : sélection de l'option via **dispatch DOM** (`mouseover/mousedown/mouseup/click`) sur
  l'élément `[role="option"]` contenant « Define below ».
- Expression sans `}}` en trop : taper d'abord `{{` (fermeture auto), puis **l'intérieur seulement**
  `$json.sessionId || $json.chat_id || "aftrsn"`, puis vérifier la valeur exacte via lecture DOM.
- Commit fiable : cliquer hors du champ + **le bouton Publish qui passe au jaune** sert de preuve que le modèle a capté la modif.

## 7. Lessons learned (pièges figés)

- **Double-déclencheur = adapter AUSSI la mémoire.** Rendre un agent appelable par `executeWorkflowTrigger` ne
  suffit pas : la **Postgres Chat Memory** en mode « Connected Chat Trigger Node » exige un champ `sessionId` que
  l'appel workflow ne fournit pas. Toujours passer sa **Session ID en « Define below »** avec une clé robuste
  couvrant les deux chemins : `{{ $json.sessionId || $json.chat_id || "<défaut>" }}`.
- **La porte de clarification passe AVANT le routage d'intent** dans le Gateway : pour un intent qui gère lui-même
  son contexte (maestro), forcer `needs_clarification=false` côté classifieur, sinon boucle sans état.
- **Éditeur d'expression n8n** : ne jamais taper la paire `{{ }}` complète (auto-fermeture). Taper `{{` puis l'intérieur.
- **Dropdowns n8n** (el-select) : si le clic pixel échoue, cibler l'option `[role="option"]` par le DOM.
- **Coût d'orchestration Maestro** : sur une vraie question stratégique, Maestro appelle actuellement TOUS ses
  sous-agents → 8 min 44 s / ~242 k tokens. Fonctionne mais à **fiabiliser/tuner** (limiter les appels). → backlog.

---
*Double sauvegarde : projet Claude « Claude Lessons » + Google Drive « LUMINA AI DOCS » (id `1rg1dzCpsI1LjOKmKEzaE0ZH83ij_i6Ot`).*
