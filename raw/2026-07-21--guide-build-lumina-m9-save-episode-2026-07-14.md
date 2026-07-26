---
type: raw
title: "GUIDE-BUILD_LUMINA-M9-Save-Episode_2026-07-14"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/guide-build_lumina-m9-save-episode_2026-07-14.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# 🔧 GUIDE-BUILD — LUMINA M9 · Save Episode (journalisation systématique au niveau Gateway)

**Version :** v0.1 (fiche de cadrage — à confirmer avant mutation de la Gateway de prod)
**Date :** 2026-07-14 · **Projet :** LUMINA OS · Personal Assistant Bot « Lyra » (Telegram) · **Module :** M9
**Boussoles :** `SPEC-BUILD_LUMINA-TELEGRAM-GATEWAY-INTENT-ROUTER` (v0.3) + `BIBLE_LUMINA-OS_360` (v2.5 → addenda → v2.9).
**Réfère à :** passation `PASSATION_LUMINA-Telegram-Lyra_M8-fait-doc-et-M9_2026-07-14.md`.

> **Portée** : journaliser **systématiquement chaque échange** (message user + réponse finale du bot + métadonnées) en **mémoire épisodique**, directement au niveau de la Gateway. Complète — sans remplacer — les épisodes « résumé-décision » que le Maestro écrit déjà pour le chemin business. Objectif : un **transcript complet et interrogeable** de la vie du bot (collection `episodic`), base des insights nocturnes.

---

## 1. Décisions de cadrage (validées Karter, 2026-07-14)

1. **Journaliser l'échange COMPLET** : message user + réponse finale du bot + métadonnées (`intent`, `brand`, chemin emprunté, `chat_id`, `message_id`, horodatage). → nécessite de faire **converger toutes les branches de réponse** vers un node de journalisation.
2. **Tout au Gateway, source distincte** : journaliser **tous** les échanges avec **`source='gateway-lyra'`**. Coexiste avec les épisodes du Maestro (résumés-décision, `source` propre) ; **pas de collision** `content_hash` (md5) car le contenu diffère.
3. **Banque = `brand` du Router** (défaut **`aftrsn`** si absent) — cohérent avec Maestro et `memory-search`.
4. **Fire-and-forget** (principe non négociable) : la journalisation ne doit **jamais** retarder ni casser la réponse à l'utilisateur. Node de save placé **en aval de l'envoi** de la réponse, en **continue-on-fail**.

---

## 2. Cible réutilisée & contrat réel (n8n fait foi — re-vérifier via store Pinia avant câblage)

- **LUMINA-MEMORY-WRITE / Save Episode** = **`zu4jfZbmDz8trQLl`** (publié, sous-workflow).
  - **Déclencheur = `executeWorkflowTrigger`** (⚠️ **PAS** un webhook : sur ce n8n en mode queue, les webhooks créés par API ne s'enregistrent pas — piège figé). → on l'appelle via un node **Execute Sub-workflow** (`Wait for completion` ON), **jamais** en HTTP.
  - **Contrat d'entrée** : `{ brand, title, content, collection, knowledge_type, source, source_ref }`.
  - Chaîne interne : `Embed (OpenAI text-embedding-3-small, 1536)` → `Build Insert` (INSERT paramétré, **`md5(content)` = idempotence**) → `Postgres Insert` → `Result`.
  - Écrit dans la banque `<brand>_memory`, `collection='episodic'` par défaut.
- **Gateway** = **`uGF3pSv6aTK79cqi`** (59 nodes après M8). C'est là qu'on ajoute M9 (additif).
- **memory-search** (pour la recette) = `POST https://n8n.aftersunpeople.com/webhook/memory-search`, body `{question, brand, limit, collection:"episodic"}` (⚠️ `episodic` **exclu par défaut** → il FAUT passer `collection:"episodic"` pour relire un épisode).
- **Error Workflow (Sentinel)** = `xzaH0uWy0idKVphF` (reste branché ; mais M9 en continue-on-fail = pas d'alerte sur échec de journalisation, cf. §6).

---

## 3. Contenu de l'épisode (mapping)

| Champ | Valeur |
|---|---|
| `brand` | `{{ brand du Router }}` sinon `"aftrsn"` |
| `collection` | `"episodic"` |
| `knowledge_type` | `"episode"` |
| `source` | `"gateway-lyra"` |
| `source_ref` | `"tg:<chat_id>:<message_id>@<ISO timestamp>"` |
| `title` | `"Lyra <intent> — <45 premiers car. du message user>"` |
| `content` | bloc structuré (ci-dessous) |

**`content` (gabarit) :**
```
Demande: <texte message user>
Contexte: intent=<intent> · brand=<brand> · chemin=<path> · chat_id=<chat_id> · <ISO ts>
Réponse: <texte réponse finale envoyée par le bot>
```
> Résumé court et factuel (pas de PII inutile ; le message user est déjà « privé bloqué » côté Router pour les tâches sensibles). Le `content` sert de trace + matière à consolidation nocturne.

---

## 4. Architecture — ajout additif dans la Gateway

### 4.A Capture de la requête (référence stable)
Le message user + `chat_id`/`message_id` sont lisibles depuis le **Telegram Trigger** (`$('<Trigger>').item…`) ; `intent`/`brand` depuis le **contrat Router** (node Set existant). Aucune capture supplémentaire nécessaire tant que ces nodes restent référençables en fin de flux.

### 4.B Convergence des réponses → journalisation
Deux nodes ajoutés, en **queue** de chaque branche de réponse **après** l'envoi Telegram :

```
[chaque node d'ENVOI de réponse terminal]
   → M9 - Build episode   (Code : assemble brand/title/content/source_ref à partir de
                            $('<Trigger>'), du contrat Router, et du reply_text de l'item entrant)
   → M9 - Save Episode     (Execute Sub-workflow → zu4jfZbmDz8trQLl, Wait ON, CONTINUE ON FAIL)
   → FIN
```

**Nodes de réponse à raccorder** (liste à **confirmer live** via le store — le Gateway a plusieurs terminaisons) :
- réponse « stub » / réponse directe,
- réponse **clarification**,
- réponse **memory-search** (M3),
- réponse **web / Hermes** (M4),
- réponse **Gmail** lecture (M5),
- réponse **Calendar** lecture (M6),
- réponse **Maestro** (chemin business),
- (M8) **carte de validation** = à **exclure** du MVP M9 (c'est une action, pas un échange conversationnel ; journalisable plus tard en `action_type`).

> **Détermination du `reply_text`** : `M9 - Build episode` lit le texte de réponse de façon **défensive** : `reply_text ?? text ?? output ?? answer ?? message` de l'item entrant. Si aucune réponse texte trouvée → titre générique, `content` sans « Réponse ».

### 4.C Pourquoi en aval de l'envoi
La réponse est **déjà partie** quand M9 s'exécute → même si `Save Episode` échoue ou traîne, l'utilisateur a sa réponse. `Continue On Fail` garantit qu'aucune erreur de journalisation ne remonte à l'utilisateur ni ne casse la branche.

---

## 5. Méthode d'édition (résiliente — pièges figés rappelés)

- Édition Gateway via **store Pinia** dans l'onglet n8n piloté : `wf = [...$pinia._s.values()].find(s=>s.$id==='workflows')` ; lire `wf.workflow.nodes/.connections`, cloner, muter (`setNodes`/`setConnections`), puis **salir par un VRAI drag sur un node réel** → bouton **« Publish »** orange → Publier → **reload + re-vérif store** (état publié fait foi ; `stateIsDirty` non fiable).
- **Multi-select canvas KO en auto** → câbler **connexion par connexion** via le store (plusieurs branches → un même `M9 - Build episode`).
- **Execute Sub-workflow** : `Wait for completion` ON ; sous-wf appelé par `executeWorkflowTrigger` **invisible en MCP** — inspecter via le store. Mapper le contrat en entrée.
- **Injection JS = base64/ASCII** (`atob`), sources sans accents ; **restreindre les retours** (filtre anti-secret « Cookie/query string »).
- Ne PAS journaliser via HTTP le webhook `zu4jfZbmDz8trQLl` (ne s'enregistre pas en queue) → Execute Sub-workflow.

---

## 6. Points de vigilance / arbitrages

- **Continue-on-fail vs Sentinel** : en continue-on-fail, un échec de journalisation **n'alerte pas** le Sentinel. Choix assumé (la journalisation ne doit pas polluer l'UX). Option future : brancher une sortie d'erreur silencieuse vers un compteur/log léger.
- **Volume** : 1 épisode par échange → la banque `aftrsn_memory/episodic` grossit. `md5(content)` évite les doublons exacts ; la consolidation nocturne condense. Purge = non (politique « garde tout »). À surveiller si le volume explose (raffinement : TTL/archive).
- **PII** : ne pas écrire de contenu sensible en clair. Le Router bloque déjà les tâches `private` en amont ; M9 journalise le message tel qu'affiché. Si besoin, filtrer/masquer dans `M9 - Build episode`.
- **Double-comptage business** : accepté (décision §1.2). Épisode Gateway (`source='gateway-lyra'`, transcript) + épisode Maestro (`source` propre, résumé-décision) coexistent.

---

## 7. Tests de recette

| # | Scénario | Attendu |
|---|---|---|
| T1 | Question mémoire simple (M3) | Réponse normale **+** 1 épisode écrit (`source='gateway-lyra'`, `collection='episodic'`, brand correct) |
| T2 | Relecture de l'épisode | `memory-search {brand, collection:"episodic", question:<extrait>}` → l'épisode ressort |
| T3 | Échange chemin business (Maestro) | Réponse normale + épisode Gateway écrit **en plus** de l'épisode Maestro (2 sources distinctes) |
| T4 | `Save Episode` en échec simulé | Réponse à l'utilisateur **inchangée** (continue-on-fail) ; aucune erreur visible |
| T5 | Non-régression M3→M8 | Toutes les réponses + la porte M8 inchangées |
| T6 | Contenu épisode | `title`/`content` bien formés (demande + contexte + réponse), `source_ref` traçable |

**DoD** : T1–T6 verts. (T4 = vérifier que la journalisation est bien non-bloquante.)

---

## 8. Ordre de build

1. [ ] Inspecter la Gateway via store Pinia : **énumérer les nodes de réponse terminaux** réels + vérifier la référençabilité du Trigger et du contrat Router en fin de flux.
2. [ ] Créer `M9 - Build episode` (Code) : assemble le contrat `{brand,title,content,collection,knowledge_type,source,source_ref}`.
3. [ ] Créer `M9 - Save Episode` (Execute Sub-workflow → `zu4jfZbmDz8trQLl`, Wait ON, **Continue On Fail**).
4. [ ] Câbler chaque node de réponse terminal → `M9 - Build episode` → `M9 - Save Episode` (connexion par connexion via store).
5. [ ] Publier la Gateway ; **reload + re-vérif store**.
6. [ ] Tests T1–T6 depuis Telegram + relecture `memory-search collection=episodic` ; non-régression M3→M8.
7. [ ] Doc convention §3 : POS-EXACT + POS-GENERIQUE (Difficultés / Solutions / Lessons), double-save Claude + Drive, addendum **Bible v3.0**, cocher Spec Build, MàJ mémoire (`lumina-m9-save-episode`).

---

## 9. Difficultés rencontrées / Solutions / Lessons learned
*(à compléter pendant le build — sections obligatoires par convention §3)*

- **Difficultés rencontrées** : —
- **Solutions implémentées** : —
- **Lessons learned** : —

---

*GUIDE-BUILD M9 v0.1 — 2026-07-14. Design verrouillé sur les 4 décisions §1. n8n fait foi : re-vérifier IDs/contrats en live (store Pinia) avant de câbler. À confirmer par Karter avant mutation de la Gateway de production.*
