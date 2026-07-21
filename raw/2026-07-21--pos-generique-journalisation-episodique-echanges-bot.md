---
type: raw
title: "POS-GENERIQUE_Journalisation-episodique-echanges-bot"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_journalisation-episodique-echanges-bot.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GENERIQUE — Journalisation épisodique systématique des échanges d'un bot (n8n)

**Objet :** patron réutilisable pour tracer **chaque échange** d'un bot conversationnel (message + réponse + métadonnées) en **mémoire épisodique**, au niveau de la Gateway, sans jamais dégrader l'expérience (fire-and-forget). Base d'un transcript interrogeable et de la consolidation en insights.
**Pré-requis :** Gateway n8n avec un node d'écriture mémoire (sous-workflow) réutilisable + une banque par entité (`<brand>_memory` ou équivalent).

---

## 1. Principe

Une seule règle : **journaliser en aval de l'envoi de la réponse**, jamais avant. Ainsi la réponse est déjà partie ; la journalisation qui suit ne peut ni la retarder ni la casser. On y ajoute `continue-on-fail` pour qu'un échec d'écriture reste invisible pour l'utilisateur.

Deux nodes additifs :
- **Build episode** (Code) : assemble le contrat d'épisode depuis des références stables (message d'origine, sortie du classifieur, réponse de l'item entrant).
- **Save episode** (Execute Sub-workflow) : appelle le writer mémoire, `wait` ON, **`onError=continueRegularOutput`**.

Les deux sont branchés **après chaque node de réponse terminal** (convergence).

---

## 2. Contrat d'épisode (gabarit)

```
brand           : <entité du classifieur>, normalisé (voir §5) → défaut sûr
collection      : "episodic"
knowledge_type  : "episode"
source          : "<bot>-<canal>"          // distingue des autres écrivains (ex. orchestrateur)
source_ref      : "<canal>:<chat>:<msg>@<ISO ts>"
title           : "<bot> <intent> - <extrait message>"
content         : "Demande: <msg user>\nContexte: intent/brand/chemin/chat/ts\nReponse: <réponse>"
```
Idempotence : le writer hashe `content` (md5) → pas de doublon exact.

---

## 3. Build episode (Code, points clés)

- **Message user + ids** : lire depuis le node de normalisation d'entrée (`text`, `chat_id`, `message_id`).
- **intent / brand** : lire depuis la sortie du classifieur/routeur.
- **réponse** : lire l'item entrant de façon **défensive** (`result.text ?? text ?? message ?? answer`), car chaque node de réponse a une forme différente.
- **chemin** : `$prevNode.name` = le node de réponse qui a déclenché la journalisation.
- **normaliser le brand** (§5) avant émission.

## 4. Save episode (Execute Sub-workflow, points clés)

- Cloner un node Execute Sub-workflow **existant qui marche** (schéma garanti), puis remplacer l'`workflowId` par le writer mémoire.
- `wait for completion` ON ; **`onError=continueRegularOutput`**.
- **Passthrough** : si le trigger du writer « accepte toutes les données » (lit `$json`), un mapping `defineBelow` vide suffit — l'item de Build episode transite tel quel.
- Si le writer n'est pas un webhook (piège n8n en mode queue : les webhooks créés par API ne s'enregistrent pas) → l'appeler en **Execute Sub-workflow** (`executeWorkflowTrigger`), pas en HTTP.

## 5. Câblage & convergence

Brancher **chaque node de réponse terminal conversationnel** → Build episode → Save episode. **Exclure** : messages système (refus d'accès, aide statique), **clarifications** (échange incomplet), et **actions** (validations, confirmations) — à journaliser séparément en `action_type` si audit voulu.

Édition résiliente : mutations via le store (`addNode`/`addConnection`), puis **VRAI drag** sur un node réel pour marquer `dirty`, Publish, re-vérif via store.

---

## 6. Difficultés rencontrées / Solutions / Lessons learned

### Difficultés rencontrées
1. **Écriture vers une banque inexistante** : le classifieur renvoie une valeur d'entité « nulle » (ex. `none`) pour une requête générale → le writer dérive un nom de table inexistant.
2. **Publish resté inactif** après édition programmatique du code (pas de `dirty`).
3. **Clarification multi-tours** : le tour de clarification n'est pas un échange complet.

### Solutions implémentées
1. **Normaliser l'entité** dans Build episode (valeur nulle/vide → défaut sûr) avant l'écriture.
2. **Interaction DOM réelle** (drag) pour activer Publish.
3. Ne journaliser que le tour résolu (complet) ; clarification exclue.

### Lessons learned (pièges figés)
- **Journalisation en fire-and-forget** (aval de l'envoi + `continue-on-fail`) : un bug d'écriture apparaît dans les logs **sans jamais toucher l'UX**. C'est le comportement cible, pas un défaut.
- **Un writer qui dérive une ressource d'un champ d'entrée → normaliser ce champ** ; une erreur aval « ressource inexistante » désigne la **valeur d'entrée**, pas le câblage.
- **Passthrough** : reproduire un node d'appel existant qui fonctionne plutôt que deviner le mapping d'entrée.
- **Mutation store ≠ dirty** : seule une interaction *trusted* active la publication.

---

## 7. Tests de recette (gabarit)

| # | Scénario | Attendu |
|---|---|---|
| T1 | Échange normal | Réponse + 1 épisode (`source`, `collection=episodic`, bonne banque) |
| T2 | Relecture | recherche mémoire `collection=episodic` → l'épisode ressort |
| T3 | Échec d'écriture simulé | Réponse **inchangée** ; workflow « Succeeded » |
| T4 | Non-régression | Toutes les réponses inchangées |

---

*POS-GENERIQUE — Journalisation épisodique des échanges. Patron dérivé du build LUMINA M9 (Telegram « Lyra »). Rejouable sur tout bot et toute entité : capturer l'échange complet, en aval de l'envoi, en fire-and-forget.*
