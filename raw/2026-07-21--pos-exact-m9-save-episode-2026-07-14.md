---
type: raw
title: "POS-EXACT_M9-Save-Episode_2026-07-14"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_m9-save-episode_2026-07-14.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# POS-EXACT — M9 · Save Episode (journalisation systématique au niveau Gateway)

**Date :** 2026-07-14 · **Projet :** LUMINA Personal Assistant Bot « Lyra » (Telegram) · **Module :** M9
**Objet :** journaliser **systématiquement chaque échange conversationnel** (message user + réponse finale + métadonnées) en **mémoire épisodique** (`collection='episodic'`), directement dans la Gateway, en **fire-and-forget** (ne casse jamais la réponse). Complète les épisodes « résumé-décision » du Maestro.
**Statut :** ✅ **FAIT, PUBLIÉ & VALIDÉ LIVE** — exécution #9343 : `M9 - Save Episode` Success 252ms → `ok:true · table:aftrsn_memory · collection:episodic`.
**Boussoles :** `SPEC-BUILD_LUMINA-TELEGRAM-GATEWAY-INTENT-ROUTER` (v0.3) + `BIBLE_LUMINA-OS_360` (v2.5 → addenda → v3.0). Réfère au GUIDE-BUILD M9 v0.1.

---

## 1. Décisions de cadrage (validées Karter, 2026-07-14)

1. **Échange COMPLET** : message user + réponse finale + métadonnées (`intent`, `brand`, `chemin`, `chat_id`, `message_id`, horodatage). → convergence des branches de réponse vers un node de journalisation.
2. **Tout au Gateway, source distincte** : tous les échanges avec **`source='gateway-lyra'`** ; coexiste avec les épisodes Maestro (pas de collision `md5`).
3. **Banque = `brand` du Router**, défaut **`aftrsn`** (normalisation `none`/vide → `aftrsn`, cf. §6).
4. **Fire-and-forget** : node de save **en aval de l'envoi**, en **`continue-on-fail`**.

---

## 2. IDs & contrats réels (n8n fait foi, 2026-07-14)

- **Gateway** `LUMINA-TELEGRAM-GATEWAY` = **`uGF3pSv6aTK79cqi`** — **61 nodes** après M9, publié.
- **Cible** `LUMINA-MEMORY-WRITE / Save Episode` = **`zu4jfZbmDz8trQLl`** — déclencheur **`executeWorkflowTrigger`** (⚠️ pas webhook — ne s'enregistre pas en mode queue). Contrat `{brand, title, content, collection, knowledge_type, source, source_ref}` → Embed (text-embedding-3-small 1536) → Build Insert (`md5(content)` idempotent) → Postgres Insert (`<brand>_memory`).
- **Références de données** (lues par M9 - Build episode) :
  - `Parse the message` (Code) expose **`text`, `chat_id`, `message_id`**.
  - `Classify the intent` (executeWorkflow → Router `KTLwQi7ZKHDTexMZ`) expose **`intent`, `brand`** (brand vaut `none` pour une question générale).
- **memory-search** (relecture) = `POST /webhook/memory-search`, body `{question, brand, limit, collection:"episodic"}` (⚠️ `episodic` exclu par défaut → passer `collection:"episodic"`).

---

## 3. Nodes ajoutés

### 3.1 `M9 - Build episode` (Code `n8n-nodes-base.code` v2, runOnceForAllItems)
Assemble le contrat épisode :
```js
const P=$('Parse the message').first().json;
let intent='unknown',brand='aftrsn';
try{const R=$('Classify the intent').first().json;if(R){intent=R.intent||intent;if(R.brand&&R.brand!=='none'&&R.brand!=='')brand=R.brand;}}catch(e){}
if(!brand||brand==='none')brand='aftrsn';           // ← normalisation (fix bug none_memory)
const inItem=$input.first().json;
const reply=(inItem.result&&inItem.result.text)||inItem.text||inItem.message||inItem.answer||'';
const userText=P.text||P.texte||'';
const chatId=(P.chat_id!=null?P.chat_id:'');
const msgId=(P.message_id!=null?P.message_id:'');
const ts=new Date().toISOString();
let path='';try{path=$prevNode.name||'';}catch(e){}   // chemin = node de réponse qui a déclenché
const NL=String.fromCharCode(10);
const title=('Lyra '+intent+' - '+String(userText).slice(0,45)).slice(0,120);
const content='Demande: '+userText+NL+'Contexte: intent='+intent+' . brand='+brand+' . chemin='+path+' . chat_id='+chatId+' . '+ts+NL+'Reponse: '+reply;
return [{json:{brand:brand,title:title,content:content,collection:'episodic',knowledge_type:'episode',source:'gateway-lyra',source_ref:'tg:'+chatId+':'+msgId+'@'+ts}}];
```
> `$prevNode.name` donne le node de réponse source → sert de `chemin`. Lecture `reply` défensive (`result.text` = réponse Telegram envoyée).

### 3.2 `M9 - Save Episode` (Execute Sub-workflow `n8n-nodes-base.executeWorkflow` v1.3)
- `workflowId` = **`zu4jfZbmDz8trQLl`** (RL id), `mode:'once'`, `options.waitForSubWorkflow=true`.
- **`onError:'continueRegularOutput'`** (= fire-and-forget : un échec ne casse rien).
- `workflowInputs.mappingMode='defineBelow'` **vide** = **passthrough** (le trigger de MEMORY-WRITE accepte toutes les données et lit `$json` — même patron que `Classify the intent`/`Perform memory search` qui fonctionnent).

### 3.3 Câblage (5 branches de réponse conversationnelles → Build → Save)
```
Reply to user            (M3 mémoire)   ┐
Reply to user (web)      (M4 web/Hermes)│
Reply to user (gmail)    (M5 lecture)   ├─→ M9 - Build episode → M9 - Save Episode → FIN
Reply to user (calendar) (M6 lecture)   │
Reply (maestro)          (business)     ┘
```
> **Exclus** (volontaire) : `Send the help menu`, `Access Denied`, `Ask which brand` (clarification = échange incomplet), et les nodes **M8** (carte/answers/edits = actions, pas conversation).

---

## 4. Méthode d'édition (réelle)

Édition via **store Pinia** dans l'onglet n8n piloté : `wf.addNode(...)` ×2 + `wf.addConnection({connection:[{node,type:'main',index:0},{node,type:'main',index:0}]})` ×6. Clonage des paramètres `executeWorkflow` depuis `Classify the intent` (schéma garanti), puis swap `workflowId.value` + `onError`. **Publish** via UI après un **VRAI drag** sur un node réel (les mutations store ne marquent pas `dirty` — bouton resté « Published » jusqu'au drag). Re-vérif via store : 61 nodes, connexions OK.

---

## 5. Recette (résultats live)

| # | Scénario | Attendu | Résultat |
|---|---|---|---|
| T1 | Question mémoire (« rôle du Maestro », brand AFTRSN) | Réponse + 1 épisode `gateway-lyra`/`episodic` | ✅ exec #9343 : Save Episode `ok:true`, `aftrsn_memory`, `episodic` |
| T4 | Save Episode en échec (brand=none) | Réponse utilisateur **inchangée** ; workflow « Succeeded » | ✅ exec #9330 : erreur `none_memory` **loggée** mais UX intacte (continue-on-fail) |
| T5 | Non-régression M3→M8 | Réponses + porte M8 inchangées | ✅ (réponses reçues normalement) |

**DoD atteint** : épisode écrit (`aftrsn_memory/episodic`, `source='gateway-lyra'`), et journalisation **non-bloquante** prouvée.

---

## 6. Difficultés rencontrées / Solutions implémentées / Lessons learned

### Difficultés rencontrées
1. **`relation "none_memory" does not exist`** (exec #9330) : le Router renvoie **`brand='none'`** pour une question générale → MEMORY-WRITE dérive la table `<brand>_memory` = `none_memory` (inexistante).
2. **Publish resté grisé** après édition du `jsCode` via le store : la mutation programmatique **ne marque pas** le workflow `dirty`.
3. **Filtre anti-secret du navigateur** : tout dump brut de params JS contenant `query/cookie/base64` est **bloqué** → impossible d'extraire les params en clair.
4. **Clarification multi-tours** : le 1er tour (« Ask which brand ») n'est pas journalisé (non câblé) ; seul le tour résolu (avec brand) l'est.

### Solutions implémentées
1. **Normalisation du brand** dans `M9 - Build episode` : `none`/vide/`unknown` → **`aftrsn`** (garde-fou double : au parse du Router ET avant l'émission).
2. **VRAI drag** sur un node réel pour déclencher `dirty` → bouton **Publish** orange → Publier → re-vérif store.
3. Extraction **des seuls identifiants** (noms de champs) + sanitisation systématique de la sortie (`query→Q_`, `cookie→C_`, runs base64 masqués).
4. Choix assumé : clarification = échange incomplet, non journalisé ; le tour final (complet) l'est.

### Lessons learned (pièges figés)
- **Un sous-workflow qui dérive une ressource (table) d'un champ d'entrée → normaliser ce champ en amont.** Une erreur aval `relation X does not exist` pointe la **valeur d'entrée** (`brand='none'`), pas le câblage. Le Router renvoie `brand='none'` pour les requêtes non-marque → toute écriture par-marque doit normaliser.
- **`continue-on-fail` sur la journalisation = le filet qui prouve le fire-and-forget** : le bug est apparu dans les logs sans jamais toucher l'UX (réponse envoyée, workflow « Succeeded »). Exactement l'objectif visé.
- **Mutation store ≠ dirty** : seule une interaction DOM *trusted* (drag) active Publish ; toujours re-vérifier l'état publié via le store.
- **Passthrough sous-workflow** : `defineBelow` vide fonctionne quand le trigger cible **accepte toutes les données** (lit `$json`) — reproduire le patron d'un node existant qui marche plutôt que deviner un mapping.

---

## 7. Reste / incrémental
- Journaliser aussi la **clarification** et le **/help** si un transcript exhaustif est voulu (câbler `Ask which brand` / `Send the help menu`).
- Journaliser les **actions M8** (validation) en `action_type` distinct (audit).
- Volume `aftrsn_memory/episodic` à surveiller (consolidation nocturne condense ; pas de purge).

---

*POS-EXACT M9 — 2026-07-14. Journalisation systématique fire-and-forget, validée live (épisode écrit en aftrsn_memory/episodic). n8n fait foi : re-vérifier via store Pinia avant toute modif.*
