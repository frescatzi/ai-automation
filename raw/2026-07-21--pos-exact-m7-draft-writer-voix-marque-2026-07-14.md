---
type: raw
title: "POS-EXACT_M7-Draft-Writer-voix-marque_2026-07-14"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_m7-draft-writer-voix-marque_2026-07-14.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# POS-EXACT — M7 Draft Writer (voix de marque, rédaction vérifiée via Maestro)

**Date :** 2026-07-14 · **Projet :** LUMINA Personal Assistant Bot « Lyra » (Telegram) · **Module :** M7
**Objet :** rebrancher la branche « draft » de M5 (Gmail Ops) pour que la rédaction d'un brouillon d'email passe par **AFTRSN-Maestro en mode « draft+verify »** (rédaction par la Secretary + relecture parallèle Marketing/Qualité + réconciliation + journalisation mémoire), au lieu d'un simple node LLM figé.

---

## 1. Décisions de cadrage (validées Karter)

1. **Réutiliser Maestro** (ne pas construire de rédacteur autonome parallèle) : la machinerie de vérification existe déjà.
2. **Voix hybride** : prompt paramétré B2C/B2B **+** rappel des profils de marque, **avec respect strict des guidelines** — assuré par les agents spécialistes (garde-voix + qualité), pas par une simple consigne.
3. **Vérification parallèle** : Marketing (garde-voix) **∥** Qualité-Compliance (conformité + PII). Qualité-Compliance était déjà **actif** (décision §I tranchée avant cette session).
4. **Vision Hermès (cf. mémoire)** : Hermès reste l'agent central auto-améliorant ; tous les agents migreront à terme sur sa base. **Phase 1** = la Secretary rédige (rapide/fiable) et **Hermès apprend le skill** via `Save Lesson [supervised]` dès le 1er brouillon ; **Phase 2** = migrer l'exécution de la rédaction sur Hermès une fois sa latence fiabilisée (chantier connu).

---

## 2. IDs & contrats réels (n8n fait foi, 2026-07-14)

- **M5 Gmail Ops** `LUMINA-TELEGRAM-GMAIL-OPS` = **`LUCkYnz7GhjkyCtZ`** (publié, actif, 22 nodes après M7).
- **AFTRSN-Maestro** = **`OlrQO21u178SjgBK`** — modèle **Claude Haiku 4.5** (`claude-haiku-4-5-20251001`), **appelable en sous-workflow** (node `When Executed by Another Workflow`, passthrough). Son prompt système contient déjà la boucle SÉLECTION&VÉRIFICATION + mentor Hermès + Save Episode/Lesson + « actions critiques = validation Karter ». **Non modifié** (zone rouge).
- Sous-agents appelés par Maestro : **Secretary** `FtPFOKvi2t18DFb1` (rédige, draft-only), **Marketing** `wyUEojwZjSKFRaRL` (garde-voix), **Qualité-Compliance** `zbW4p5qsBYhkfBdi` (**actif**, verdict/risques/fixes + PII).
- Mémoire : **Save Episode** (collection `episodic`) + **Save Lesson** (collection `skills`, préfixe `[supervised]`) via `LUMINA-MEMORY-WRITE` `zu4jfZbmDz8trQLl`.
- **Contrat d'entrée Maestro** (passthrough) : `{ query, chat_id }` — l'agent lit `$json.query`.

---

## 3. Flux final de la branche « draft » (M5)

```
Route op (sortie index 1 = draft)
  → Prepare Maestro brief   (Code : construit le brief + query)
  → Draft via Maestro       (Execute Sub-workflow → OlrQO21u178SjgBK, Wait for completion ON)
  → Parse draft             (Code : extrait le dernier objet JSON {subject,body} + flag ok)
  → Draft OK ?              (IF sur {{ $json.ok }} === true)
        ├─ TRUE  → Route mailbox → Create Draft B2C / B2B → Shape reply (draft)
        └─ FALSE → Shape reply (error)  ["rédaction non aboutie, réessaie" — AUCUN brouillon créé]
```

**Nodes supprimés** : `Compose draft` (chainLlm figé « chaleureux ») + `Model - draft` (lmChatOpenAi).
**Nodes ajoutés** : `Prepare Maestro brief` (Code), `Draft via Maestro` (executeWorkflow v1.2, `options.waitForSubWorkflow:true`), `Draft OK?` (if v2.2, opérateur booléen `true`), `Shape reply (error)` (Code). `Parse draft` : jsCode réécrit.

**Invariant préservé** : M5 = **brouillon uniquement**, aucun node d'envoi. L'envoi réel reste gaté **M8**.

### 3.1 `Prepare Maestro brief` (points clés du brief envoyé à Maestro)
- **RÈGLE ABSOLUE DE COMPLÉTION** : appeler *réellement* les outils (Secretary → Marketing ET Qualité), attendre les retours, réconcilier, PUIS répondre.
- **INTERDIT** : tout message d'avancement / plan / « j'attends les verdicts » (n'est pas une réponse valide).
- Boîte + ton paramétrés : B2C `hello@` chaleureux/éditorial · B2B `partners@` professionnel. `brand` lu depuis `Receive from Gateway` (défaut `aftrsn`).
- MÉMOIRE : appeler `Save Episode` en fin ; APPRENTISSAGE : `Save Lesson [supervised]` si correctifs voix/conformité réutilisables.
- **FORMAT DE SORTIE** : uniquement un objet JSON `{"subject":"...","body":"..."}`.

### 3.2 `Parse draft` (extraction robuste)
- Collecte les candidats : blocs ` ```json ``` ` **et** objets `{...}` par balayage à accolades équilibrées.
- Itère du **dernier** au premier, retient le premier objet parsable ayant `subject` + `body` (garde anti-placeholder : `body` > 15 car., ≠ `...`).
- Émet un flag **`ok`** (true si vrai brouillon extrait) consommé par `Draft OK?`.

---

## 4. Méthode d'édition (résiliente, n8n via navigateur)

L'API/MCP n8n **n'édite pas** ce scope → édition via le **store Pinia** dans l'onglet piloté :
1. `wf = [...document.querySelector('#app').__vue_app__.config.globalProperties.$pinia._s.values()].find(s=>s.$id==='workflows')`.
2. Lire l'état réel (`wf.workflow.nodes` / `.connections`), cloner (`JSON.parse(JSON.stringify)`).
3. Muter : retrait/ajout de nodes, réécriture `parameters.jsCode`, réécriture des connexions. Appliquer via **`wf.setNodes(...)` + `wf.setConnections(...)`**.
4. **Persistance** : `setNodes/setConnections` **affichent** mais ne salissent pas le draft serveur → **faire un vrai drag d'un node** (événement canvas *trusted* via l'outil computer de l'extension) → le bouton passe à **« Publish »** → **Publier**.
5. **Recharger l'URL** et **re-vérifier via le store** (l'état publié fait foi).

Versions publiées : v1 (rebranchement) → v2 (brief Save Episode/Lesson + parse robuste) → v3 (garde anti-placeholder) → v4 (brief « force completion ») → **v5 (garde-fou Draft OK?)**.

---

## 5. Difficultés rencontrées

1. **Persistance n8n** : `setNodes/setConnections` n'active pas l'autosave ; `ui.stateIsDirty` reste `false` (indicateur trompeur). Publier sans salir promeut l'ancien état.
2. **Sortie `javascript_tool` bloquée** (« Cookie/query string data ») dès qu'un objet node contenait un champ type token/webhookId.
3. **Échappement du jsCode** (apostrophes FR, guillemets, backticks) impossible à écrire proprement dans l'injection.
4. **Maestro (Haiku) « narrate-and-stop »** : en live, Maestro a renvoyé un message d'avancement (« en attente des verdicts ») sans appeler réellement les vérificateurs — **variance run-à-run** (les tests API antérieurs allaient au bout).
5. **Format de sortie** : Maestro emballe le JSON dans un bloc ` ```json ` + rapport verbeux + placeholder `{"subject":"...","body":"..."}` → une regex gourmande casserait ou capturerait le placeholder.
6. **Résultat `execute_workflow` trop volumineux** (56 k puis 205 k car.) → dépasse le budget de contexte.
7. **Effet de bord** : sur échec d'extraction, M5 créait quand même un brouillon Gmail de repli (poubelle).

## 6. Solutions implémentées

1. Après mutation store, **vrai drag ~30 px** (outil computer) pour salir → le libellé du bouton (« Publish » orange) est l'indicateur fiable, pas `stateIsDirty` → puis Publish → reload + re-vérif store.
2. **Restreindre les retours** JS aux champs strictement nécessaires (pas d'objets nodes complets).
3. **jsCode transporté en base64** (`atob(...)`), sources **ASCII sans accents** ; backticks/guillemets évités via `String.fromCharCode`.
4. **Brief « force completion »** : règle absolue + interdiction explicite des messages d'avancement (v4).
5. **Parseur robuste** : fences + balayage à accolades équilibrées, dernier objet valide, **garde anti-placeholder** (body > 15 car.).
6. **Sous-agent + jq/Read** pour analyser le gros fichier de résultat hors du contexte principal.
7. **Garde-fou `Draft OK?`** (IF sur `ok`) : si pas de vrai brouillon → `Shape reply (error)`, **aucun** brouillon Gmail créé (v5).

## 7. Lessons learned (pièges figés, transférables)

- **Réutiliser l'orchestrateur > réimplémenter** : la boucle mentor/vérification/mémoire existait déjà dans Maestro ; M7 s'est réduit à du câblage + un contrat d'instruction. **Toujours tracer l'état live avant de concevoir** (n8n était très en avance sur la doc : Haiku 4.5, Qualité déjà actif, Maestro déjà appelable en sous-wf).
- **Un LLM rapide (Haiku) orchestrateur = risque de « narrate-and-stop »** non déterministe → (a) instruction de complétion explicite, (b) **toujours un garde-fou en aval** qui distingue succès/échec et **n'exécute pas d'action** (créer un brouillon) sur une sortie non conforme. Ceinture + bretelles.
- **Ne jamais faire confiance au format de sortie d'un LLM** : parser en cherchant le **dernier** objet JSON valide et **filtrer les placeholders**.
- **Édition n8n par store Pinia** : muter puis **salir par un vrai événement canvas** avant Publish ; `stateIsDirty` non fiable → se fier au libellé du bouton. Toujours **reload + re-vérif**.
- **Injection de code dans le navigateur = base64/ASCII** pour esquiver l'échappement ; **restreindre les retours** pour éviter le filtre anti-secret.
- **Gros résultats d'exécution** : déléguer l'analyse à un **sous-agent** (jq/Read) pour préserver le contexte.
- **Mémoire/apprentissage doivent être déclenchés explicitement** dans le brief (Save Episode/Lesson) : sinon l'orchestrateur ne journalise pas pour une tâche de pure rédaction → **Hermès n'apprend pas**. C'est le levier concret de « Hermès prend du muscle ».

---

## 8. État & reste

- **Validé live 2026-07-14** : brouillon B2C propre (Objet + corps) reçu depuis Telegram ; boucle Secretary + Marketing ∥ Qualité + Save Episode + Save Lesson confirmée (exécs API 7605/7626) ; garde-fou et parse durci publiés.
- **Reste (Karter)** : supprimer le brouillon de repli de test dans `hello@` (exéc 7775) + éventuellement l'épisode/leçon de test dans les banques `aftrsn` ; test B2B `partners@` de confort ; non-régression M3–M6 rapide.
- **Suite** : **M8** (garde-fou de validation `/validate | /cancel`, avant tout envoi réel), puis **M9** (Save Episode systématique de chaque échange).

*Fichiers d'appui (outputs) : `prepare_brief.js`, `parse_draft.js`, `shape_error.js`, `M7-draft-branch-nodes.json`. n8n fait foi.*
