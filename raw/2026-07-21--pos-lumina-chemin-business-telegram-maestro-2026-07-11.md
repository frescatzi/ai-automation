---
type: raw
title: "POS-LUMINA_chemin-business-Telegram-Maestro_2026-07-11"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_chemin-business-telegram-maestro_2026-07-11.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# POS-LUMINA — Chemin business Telegram → AFTRSN-Maestro (fiabilisation)

**Type :** Procédure Opérationnelle Standard (POS formalisée)
**Scope :** LUMINA OS / AFTER SUN PEOPLE — Bot Telegram « Lyra »
**Date :** 2026-07-11
**Statut :** ✅ En production, validé bout-en-bout

---

## 1. But

Garantir qu'une **question de décision / stratégie / business** envoyée à Lyra (Telegram) est classée `intent=maestro`,
routée jusqu'à l'agent orchestrateur **AFTRSN-Maestro**, traitée (mémoire OK), et que la réponse est **renvoyée dans
Telegram** — sans boucle de clarification de marque et sans erreur de mémoire.

## 2. Préconditions

1. Accès n8n `https://n8n.aftersunpeople.com` avec droits `workflow:update` + `workflow:publish`.
2. Workflows en ligne : Router `KTLwQi7ZKHDTexMZ`, Maestro `OlrQO21u178SjgBK`, Gateway `uGF3pSv6aTK79cqi`.
3. Bot Telegram `@luminaLyraBot`, credential `LUMINA-Lyra - Telegram`, owner `chat_id 776345147`.
4. Édition via navigateur piloté en **clics SPA** (jamais de rechargement d'URL en cours d'édition).

## 3. Étapes (numérotées à partir de 1)

1. **Router — neutraliser la clarification pour maestro.** Node « Classifieur d'intention » :
   - ajouter à la règle maestro : *« Pour intent="maestro", mets TOUJOURS needs_clarification=false et brand="aftrsn"
     par défaut (Maestro gère la marque). »*
   - ajouter l'exception *« (sauf intent="maestro") »* à la règle de marque ambiguë.
   - **Publish** (modif ciblée de la valeur live uniquement).
2. **Maestro — rendre la mémoire compatible avec l'appel Gateway.** Node « Chat Memory » :
   - Session ID → **« Define below »** ;
   - Key (Expression) → `{{ $json.sessionId || $json.chat_id || "aftrsn" }}` ;
   - **Publish**.
3. **Test end-to-end.** Envoyer en Telegram une question business type
   (« Devrait-on lancer une gamme de produits solaires cet été ? »).
4. **Contrôle des exécutions.** Vérifier `AFTRSN-Maestro` (champ `output` non vide, Chat Memory sans erreur) et
   `LUMINA-TELEGRAM-GATEWAY` (node « Reply (maestro) » → `ok:true`, `message_id` renseigné).

## 4. Vérification (critères d'acceptation)

- Aucune boucle « LUMINA ou AFTRSN ? » sur une question business.
- `AFTRSN-Maestro` : exécution en succès, `output` = recommandation structurée.
- `LUMINA-TELEGRAM-GATEWAY` : « Reply (maestro) » `ok:true` → message reçu dans Telegram.
- Réf. validation 2026-07-11 : Maestro exec **3907** (succès), Gateway exec **3905** (succès, `message_id 92`).

## 5. Rollback

1. Router : restaurer la version antérieure à `f7fe2432` (ou retirer les 2 phrases ajoutées) → Publish.
2. Maestro : Session ID = « Connected Chat Trigger Node » (clé `{{ $json.sessionId }}`), ou version antérieure à
   `ca75c537` → Publish.

## 6. Contrats & IDs clés

| Élément | ID / valeur |
|---|---|
| Router (intent) | `KTLwQi7ZKHDTexMZ` — version fix `f7fe2432` |
| Maestro (orchestrateur) | `OlrQO21u178SjgBK` — sortie champ **`output`** — version fix `ca75c537` |
| Gateway (Telegram) | `uGF3pSv6aTK79cqi` — node « Reply (maestro) » |
| Clé de session Maestro | `{{ $json.sessionId || $json.chat_id || "aftrsn" }}` |
| Table mémoire | `n8n_chat_histories` (credential `LUMINA_Postgres`) |
| Credential Telegram | `LUMINA-Lyra - Telegram` (`@luminaLyraBot`) |
| owner chat_id | `776345147` |
| Drive « LUMINA AI DOCS » | `1rg1dzCpsI1LjOKmKEzaE0ZH83ij_i6Ot` |

---

## 7. Difficultés rencontrées

- Le blocage n'était pas unique : réparer le routage (Router) a **révélé un 2e blocage** en aval (Chat Memory Maestro),
  invisible tant que la question n'atteignait pas Maestro.
- Le Router n'est **pas éditable** via le connecteur n8n MCP (hors scope + lecture seule) → passage obligé par le navigateur.
- Dropdown « Session ID » et éditeur d'expression n8n **résistants** aux interactions simulées.

## 8. Solutions implémentées

- Correctif **Router** : `needs_clarification=false` forcé pour `intent=maestro` + exception marque ambiguë.
- Correctif **Maestro** : Session ID « Define below » + clé défensive couvrant chat ET appel Gateway.
- Sélection d'option via DOM et saisie d'expression en 2 temps (`{{` puis intérieur) pour contourner l'auto-fermeture.

## 9. Lessons learned (pièges figés)

- **Un agent rendu « appelable » doit voir SA MÉMOIRE adaptée** au trigger d'appel (sinon *« No session ID found »*).
- **Clarification sans état = boucle** : neutraliser la clarification pour les intents auto-gérés au niveau du classifieur.
- **Ordre des portes du Gateway** : la clarification passe avant le routage d'intent — en tenir compte.
- **Backlog fiabilisation** : Maestro appelle actuellement TOUS ses sous-agents (≈ 8 min 44 s / ≈ 242 k tokens par
  question). À tuner (garde-fou sur le nombre/sélection de spécialistes). Chantier séparé « état de conversation »
  pour fiabiliser la clarification générale (memory_search / general_question).

---
*Numérotation à partir de 1. Double sauvegarde obligatoire : projet Claude « Claude Lessons » + Google Drive « LUMINA AI DOCS ».*
