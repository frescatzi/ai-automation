---
type: raw
title: "POS-LUMINA_Fix-Knowledge-JSON-body-Maestro_2026-07-09"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_fix-knowledge-json-body-maestro_2026-07-09.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS — Fix node « Knowledge » de Maestro (body JSON cassé par caractères spéciaux) · EXACT

**Version :** v1.1
**Date :** 2026-07-09
**Projet :** LUMINA OS · AFTRSN — agent Maestro + 6 sous-agents
**Étape couverte :** correction critique du node `Knowledge` (outil memory-search) de `AFTRSN-Maestro`, **propagée aux 6 sous-agents**. Découverte lors de la vérif du contrat memory-search (préparation M3).

> **v1.1 (2026-07-09)** — ajout §9 : propagation du fix aux 6 sous-agents + incident dépublication + piège chat path.

---

## 1. Objectif

Corriger le node `Knowledge` de l'agent **Maestro** qui **plante** (`JSON parameter needs to be valid JSON`) dès que la question générée par l'IA contient un guillemet, un antislash ou un saut de ligne — ce qui rend la recherche mémoire de Maestro inutilisable dans la plupart des cas réels. Après correction : la valeur dynamique est échappée proprement et l'appel à memory-search réussit quelle que soit la question.

## 2. Prérequis

1. Workflow **`AFTRSN-Maestro`** — ID **`OlrQO21u178SjgBK`** (projet AFTRSN, **Active**).
2. Node cible : **`Knowledge`** (id `56691498-b74e-4736-952a-e9daf868b896`), type `n8n-nodes-base.httpRequestTool` v4.4, `POST https://n8n.aftersunpeople.com/webhook/memory-search`.
3. Accès éditeur n8n avec droit de **Save** sur le workflow.

## 3. Procédure pas à pas

### 3.1 — Diagnostic (état cassé)
1. Le node envoyait le body en mode **Using JSON** avec un JSON écrit à la main :
   ```
   ={ "question": "{{ $fromAI('question', 'the question to search in the central memory') }}" }
   ```
2. La valeur `$fromAI(...)` est injectée **entre guillemets, sans échappement**. Dès qu'elle contient `"`, `\` ou un retour à la ligne, le JSON résultant est invalide → `NodeOperationError: JSON parameter needs to be valid JSON`. Le node échoue **avant** d'appeler le webhook.

### 3.2 — Correction (état validé)
1. Ouvrir `AFTRSN-Maestro`, double-cliquer le node **Knowledge**.
2. Laisser *Send Body* = ON, *Body Content Type* = **JSON**, *Specify Body* = **Using JSON**.
3. Dans le champ **JSON** (déjà en mode expression `fx`), remplacer tout le contenu par — **sans `=` initial** :
   ```
   {{ JSON.stringify({ question: $fromAI('question', 'the question to search in the central memory') }) }}
   ```
   `JSON.stringify` produit un JSON valide et **échappé** quel que soit le contenu de la question. La clé `question` et la description de `$fromAI` sont conservées → le schéma de l'outil vu par l'agent ne change pas.
4. **Save** le workflow (Maestro est actif : la version en prod n'est mise à jour qu'après Save).

## 4. Vérification / recette

1. Exécuter Maestro (chat) avec une question **contenant volontairement des guillemets et un saut de ligne**, ex. : « … collection "episodic" exclue par défaut ? param "limit" ? ».
2. Attendu : le node `Knowledge` a `executionStatus = success` et memory-search renvoie des passages (forme `id / title / extrait / collection / similarite`).
3. **Résultat obtenu** — exécution n8n **`2152`** (2026-07-09), `success: true` : requête `Knowledge` avec guillemets échappés (`\"canon\", \"episodic\"…`), **5 passages** renvoyés (collections `canon` + `ai_os`), rapport Maestro produit, épisode journalisé.

## 5. Difficultés rencontrées

1. **Faux départ « Using Fields Below ».** Tentative de passer en mode champs : le wrapper `{ "question": … }` a été retiré mais le mode est resté **Using JSON**, laissant `={{ $fromAI(...) }}` → body = chaîne brute non-JSON → même erreur.
2. **Sauvegarde oubliée.** Après édition correcte dans le panneau, l'exécution lisait **toujours l'ancien body** : sur un workflow actif, un appel externe (execute/webhook) ne voit que la version **sauvegardée**.
3. **Double `=`.** Coller `={{ … }}` dans un champ **déjà** en mode expression a produit `=={{ … }}` : le `=` surnuméraire devient littéral → body `={"question":…}` invalide.

## 6. Solutions implémentées

1. Rester en **Using JSON** et confier l'échappement à **`JSON.stringify`** (plus robuste et plus court qu'un montage de champs).
2. **Save** systématique après édition, puis re-test via une exécution réelle.
3. Dans le champ, saisir le contenu **sans `=` initial** (le `=` est le préfixe interne de sérialisation d'n8n, à ne pas retaper).

## 7. Lessons learned

1. **Ne jamais construire un JSON à la main** avec une valeur dynamique (`$fromAI`, expression) injectée entre guillemets. Utiliser `JSON.stringify({...})` (ou *Using Fields Below* correctement configuré) pour laisser l'échappement se faire tout seul. → **piège figé Bible**.
2. Dans un champ n8n **déjà en mode expression** (`fx`), le contenu commence par `{{`, **pas** par `={{`.
3. **Workflow actif = Save obligatoire** avant tout re-test externe ; l'éditeur peut afficher la bonne valeur sans qu'elle soit persistée.
4. `$fromAI` est détecté quel que soit son emplacement → on peut restructurer le body sans casser le schéma de l'outil.
5. Ce patron cassé est **partagé** : les nodes `Knowledge` des sous-agents suivent le même modèle et sont à corriger de la même façon.

## 8. Références

- Bible 360° §E (Maestro) ; memory-search = `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` `ZhUeKCo8nMp35gk1` (§E.2) ; addendum piège figé **v1.8**.
- Exécutions n8n : `2137` / `2143` / `2146` / `2149` (échecs successifs) → **`2152`** (validation Maestro, 2026-07-09).
- POS générique associée : `POS-GENERIQUE_Fix-httpRequestTool-JSON-body-fromAI.md`.

---

## 9. Propagation aux 6 sous-agents (2026-07-09)

### 9.1 — Cibles et body corrigé
Les 6 sous-agents partagent le même gabarit, avec un body `Knowledge` **incluant `brand: "aftrsn"`** (différent de Maestro qui n'a que `question`). Body cassé de départ :
```
{ "question": "{{ $fromAI('question', …) }}", "brand": "aftrsn" }
```
Ligne corrigée appliquée dans chacun (champ JSON, mode expression, **sans `=` initial**) :
```
{{ JSON.stringify({ question: $fromAI('question', 'the question to search in the central memory'), brand: 'aftrsn' }) }}
```
Agents : Culture-Steward `vK6B2WuDfbse41Jv` · Experience-Designer `bBu5ycRwHoK7ACdV` · Secretary `FtPFOKvi2t18DFb1` · Marketing `wyUEojwZjSKFRaRL` · Comptable-Finance `F1wsXbsYZdEkwQ3n` · Qualité-Compliance `zbW4p5qsBYhkfBdi`.

### 9.2 — Vérification
Fan-out Maestro vers les 6 avec une question à guillemets — exécution **`2162`** (2026-07-09) : **6/6 succès**, chaque node Knowledge exécuté, extraits du canon renvoyés, aucune erreur JSON. Sous-exécutions : `2171` / `2170` / `2169` / `2167` / `2165` / `2163`.

### 9.3 — Incident & pièges rencontrés pendant l'opération
1. **Incident dépublication (à ne pas refaire).** Les 6 sous-agents ont été **dépubliés** en cours de route (sur mauvaise indication) → délégation Maestro cassée : *« Workflow is not active and cannot be executed »* (appel Maestro) et *« webhook not registered »* (chat direct). **État PROD correct = sous-agents ACTIVE/publiés.** Restaurés en Active.
2. **`Available in MCP` exige un workflow publié** (tooltip n8n) avec trigger Schedule/Webhook/Form/Chat → n'est **pas** un moyen d'inspecter un workflow inactif.
3. **Piège figé — clonage duplique le path du Chat Trigger.** Qualité-Compliance et Culture-Steward partageaient le **même** path de webhook de Chat Trigger (`…/e0eb4921-…/chat`) → *« Conflicting Chat Path »*, impossible d'activer les deux. **Correctif** : régénérer un Chat Trigger neuf (path unique) sur le clone, reconnecter à l'AI Agent, Save, Activate. À intégrer dans `cloneRoster` (régénérer un webhookId par clone).
