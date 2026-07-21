---
type: raw
title: "POS-GENERIQUE_Branche-calendar-ops-telegram"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_branche-calendar-ops-telegram.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Ajouter une branche « ops lecture seule » à un bot-passerelle Telegram (n8n)

**Patron réutilisable** (dérivé de M6 Calendar Ops). Sert à brancher un nouvel intent (ex. `calendar_read`, `crm_read`, `docs_read`…) sur un **sous-workflow d'ops dédié** en **lecture seule**, puis à l'insérer dans le dispatch d'une Gateway existante — **sans casser les branches en place**.

---

## 1. Principe

- **Sous-workflow dédié** (découplé des agents de marque) : `Receive (Execute Workflow Trigger) → Understand (LLM → plan JSON) → Parse plan (Code) → Route op (Switch read/action) → [lectures parallèles + tag + Merge + Summarize + Shape reply] ; branche action → message gaté` (aucune écriture).
- Contrat : entrée `{ intent, query, brand, chat_id, message_id }` → sortie `{ chat_id, text }` (identique aux autres modules → le node Telegram de réponse ne change pas).
- **Action critique = jamais automatique** : lecture + brouillon OK ; envoi / écriture / suppression / dépense = **gaté** par une validation humaine (module dédié). Autorisable sur **ordre explicite** de l'utilisateur.

---

## 2. Construire le sous-workflow (méthode résiliente)

1. **Configurer un node « source » une fois à la main**, puis **lire sa structure exacte via le store Pinia** (`$pinia._s → 'workflows' → workflow.nodes`) — indispensable quand le mapping JSON d'un node n'est pas évident (ex. champs top-level vs sous `options`, resourceLocator en mode `list` vs `id`).
2. **Construire le JSON complet** du sous-wf et l'**importer** (⋮ → Import from file, ou `file_upload` sur l'input caché). Y **embarquer les `credentials:{<type>:{id,name}}`** (IDs lus dans le store) → auto-liés à l'import, zéro assignation manuelle (crucial quand plusieurs credentials du même type existent : l'auto-assignation est sinon aléatoire).
3. **Tester le sous-wf en isolé** avant tout câblage : `set mock data` sur le trigger (remplir via `document.execCommand('insertText', false, JSON.stringify([...]))` pour éviter l'auto-fermeture des crochets), exécuter, vérifier la sortie ; **dépingler le mock** ensuite.
4. Régler **Error Workflow** (Sentinel) + **tags** + **Publier**.

---

## 3. Insérer la branche dans la Gateway (sans multi-select fiable)

Le **multi-select du canvas n8n est peu fiable via l'automatisation** (shift/cmd-clic n'accumulent pas ; rubber-band erratique). Deux options :

- **Manuel (UI)** : dupliquer une branche-sœur existante (IF + Send typing + Prepare + Execute Sub-wf + Reply) pour hériter des credentials + expressions validées, puis n'ajuster que l'IF (conditions), le nom du Prepare et la cible du Execute Sub-workflow ; recâbler la sortie FALSE de la branche précédente vers le nouvel IF, et le FALSE du nouvel IF vers la suite.
- **Programmatique (store) — le plus fiable en auto** :
  1. Lire `wf = [...app.__vue_app__.config.globalProperties.$pinia._s.values()].find(s=>s.$id==='workflows')`.
  2. **Cloner** les nodes de la branche-sœur (`JSON.parse(JSON.stringify(node))`, `id=crypto.randomUUID()`, positions offset), ajuster (conditions IF, `workflowId.value`, credentials si besoin).
  3. `wf.setNodes([...existants, ...clones])` + `wf.setConnections(nouvellesConnexions)` (recâbler : `FALSE de la branche précédente → nouvel IF` ; `nouvel IF TRUE → premier node de la branche` ; `nouvel IF FALSE → node suivant historique`).
  4. **Vérifier via le store** (nodeCount, connexions, credentials) — le canvas se re-render depuis le store.

> **⚠️ PERSISTANCE (piège majeur)** : `setNodes/setConnections` **affiche** sur le canvas mais **ne sauvegarde pas** le draft serveur → **Publish promeut l'ancien état**. **FIX** : après la mutation, déclencher un **vrai événement canvas** (léger drag d'un node ~20px) → l'autosave écrit le draft (`ui.stateIsDirty → false`), **puis** Publier. **Toujours recharger + re-vérifier via le store** (seule preuve fiable). Publier crée une version de rollback.

---

## 4. Difficultés rencontrées (génériques)

- Mapping JSON d'un node tiers inconnu → mauvais champs silencieusement ignorés (valeurs par défaut conservées).
- Import qui « perd » un node lors d'éditions parasites ; duplication canvas + câblage au pixel peu fiables.
- Éditeurs CodeMirror n8n qui auto-ferment `{{ }}` `()"'`.
- Multi-select canvas non fonctionnel en automatisation.
- Mutation store non persistée sans événement UI déclencheur.
- `Send typing` (sendChatAction) sur une credential bot ≠ celle du `Reply` → indicateur « typing » dans le mauvais chat.

## 5. Solutions / Lessons learned (génériques)

- **Le store Pinia = source de vérité** pour lire (structure, IDs, credentials, connexions) et écrire (`setNodes`/`setConnections`) quand le canvas ne coopère pas.
- **Import avec credentials embarqués** = fiabilité + gain de temps.
- **`execCommand('insertText')`** = anti-auto-fermeture (champs + mock data).
- **Persistance = mutation store → vrai événement canvas (autosave) → publish → reload+revérif.**
- **`Send typing` doit utiliser la même credential bot que le `Reply`.**
- **Toujours re-tracer le câblage réel en live avant d'insérer** (n8n fait foi) ; vérifier chaque credential ; Execute Sub-workflow avec **Wait for completion = ON** ; `alwaysOutputData` ON sur les lectures ; sortie Telegram **Parse Mode = None** + attribution OFF.

---

*POS-GÉNÉRIQUE — 2026-07-13. Double-sauvegardé : projet Claude « Claude Lessons » (`Marches-a-suivre/`) + Google Drive `LUMINA AI DOCS/POS-GENERIQUE`.*
