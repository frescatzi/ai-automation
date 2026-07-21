---
type: raw
title: "POS-GENERIQUE_basculer-runner-triggers-notion-vers-scan-planifie-et-durcir-multi-items_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_basculer-runner-triggers-notion-vers-scan-planifie-et-durcir-multi-items_2026-07-18.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Basculer un runner n8n de triggers Notion vers un scan planifié, et le durcir en multi-items

**Date :** 18.07.2026 · **Type :** POS-GÉNÉRIQUE (patron réutilisable) · **Contexte d'origine :** fiabilisation M3/M4 du moteur événementiel AFTRSN.

**But :** rendre un runner n8n déclenché par des Notion Triggers **fiable et reprenable**, en le passant sur un **scan planifié + requête d'état**, sans introduire de bug de traitement multi-items ni de boucle d'écriture répétée.

---

## Quand appliquer
- Un runner dépend d'un **Notion Trigger « Updated »** (ou « Added ») et rate des **reprises** (une ligne repassée en To Do ne re-déclenche pas).
- Cause racine : le trigger « Updated » **dé-duplique par page** (`staticData.possibleDuplicates`) → il ne remord pas sur une même page déjà vue. Voir la leçon de fiabilité du moteur.

## Patron cible (identique à M1/M2)
```
[Schedule Trigger 1 min] → [Notion getAll (returnAll) sur la DB source] → [Code "faut-il agir ?"] → … aval inchangé …
```
Le Code « faut-il agir ? » **boucle sur `$input.all()`**, filtre par état réel (statut, exécuteur, présence d'une relation…), et **émet un item par ligne éligible**.

## Marche à suivre (par API n8n-mcp)
1. `n8n_get_workflow` mode `structure` sur le runner **et** sur un runner déjà en scan (référence, ex. M2) ; lire les nœuds pilotes en mode `filtered`.
2. `update_partial_workflow` avec `validateOnly:true` d'abord :
   - `removeNode` les 2 Notion Triggers **et** le « Read the X » (le `getAll` renvoie déjà la page complète, propriétés `property_snake_case` incluses — plus besoin d'un `get` unitaire).
   - `addNode` un `scheduleTrigger` (1 min) + un node Notion `getAll` (`returnAll:true`) sur la DB source (même credential).
   - `addConnection` Schedule → getAll → Code « faut-il agir ? ».
3. Appliquer, puis **traiter le point multi-items** (voir ci-dessous), `n8n_validate_workflow`, et vérifier le graphe **publié** (`mode:'active'`).
4. Vérifier par **exécutions réelles** (`n8n_executions`) : compter les `itemsOutput` de chaque nœud clé ; ne jamais se fier au seul « success ».

---

## Le point multi-items (le vrai piège) — décider selon que le runner mute ou non sa source

### Difficultés
- En trigger, le Code pilote reçoit **1 item** → les nœuds aval qui lisent `$('NœudAmont').first()` marchent par accident.
- En scan, le `getAll` peut rendre **N lignes éligibles** dans un tick → `.first()` ne traite que la 1ʳᵉ ; les autres sont **silencieusement perdues**. Un nœud Code « Run once for all items » qui `return [{json:…}]` unique **collapse** aussi vers le 1ᵉʳ.
- Piège inverse : un scan sur une source **jamais mutée** ré-exécute l'aval (ré-écriture / re-PATCH) **à chaque minute**, indéfiniment.

### Solutions
- **Si le runner MUTE la ligne qui l'éveille** (ex. `Status → To Validate`) et que le Code pilote **filtre sur ce statut** : la liste **rétrécit** à chaque drainage → garder `.first()` est correct (1 item/tick, comme M2). Rien à durcir dans le tail.
- **Si le runner NE MUTE PAS** la ligne (idempotence par requête d'existence) : la liste **ne rétrécit pas** → il **FAUT** un vrai traitement multi-items dans les nœuds aval Code :
  - boucler sur `$input.all()` et aligner les contextes **par index** : `const ctx = $('NœudAmont').all()[i].json;` (l'ordre est préservé à travers les nodes Notion/HTTP « once per item »),
  - pousser `pairedItem:{item:i}` sur chaque sortie.
- **Si la source du scan n'est pas la ligne mutée** (ex. scan sur la copie, mais on mute la tâche) : ajouter un **garde-fou de drainage** — n'agir que si une requête d'état trouve encore une ligne « à faire » (ex. tâche `To Do`) ; sinon `continue` pour écarter l'item. Cela empêche la ré-écriture répétée, et le multi-items en tête empêche qu'une ligne déjà drainée en masque une autre.

### Lessons
- **Ne jamais dépendre du trigger Notion « Updated »** pour une reprise. Chemins fiables : `Added` (page neuve) **ou** scan planifié + requête d'état.
- **`.first()` est un piège en scan.** Le remplacer par un parcours **indexé** (`.all()[i]`) dès que le nœud amont peut émettre plusieurs items ; l'alignement par index est plus robuste que le `pairedItem` walking à travers des branches.
- Le besoin de multi-items **dépend du modèle de drainage** : mutation-de-la-source ⇒ `.first()` OK (auto-drainage 1/tick) ; pas-de-mutation ⇒ multi-items obligatoire ; source ≠ objet muté ⇒ garde-fou de drainage.
- **Modifier le moins possible le tail à risque.** Quand l'écriture externe (Wix, etc.) est déjà validée, ne toucher que le front-end scan + les 2 nœuds de tête, et laisser le tail intact (blast radius minimal).
- **Preuve par exécutions**, pas par validation statique : comparer `itemsOutput` avant/après ; provoquer un cas actif contrôlé et réversible (ex. remettre 1 ligne en To Do) pour prouver le chemin actif **et** le re-drainage (absence de ré-écriture).
- Toujours vérifier le **graphe publié** (`mode:'active'`) après un patch : n8n a un modèle draft/publish.
