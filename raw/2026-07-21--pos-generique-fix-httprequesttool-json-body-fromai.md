---
type: raw
title: "POS-GENERIQUE_Fix-httpRequestTool-JSON-body-fromAI"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_fix-httprequesttool-json-body-fromai.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS — Fix body JSON d'un node httpRequestTool alimenté par une valeur IA/expression · GÉNÉRIQUE

**Version :** v1.0
**Objet :** patron réutilisable — sécuriser le body JSON d'un `httpRequestTool` (ou HTTP Request) dont une valeur provient de `$fromAI` / d'une expression, pour qu'il ne casse pas sur les caractères spéciaux.

---

## 1. Objectif

Empêcher l'erreur `JSON parameter needs to be valid JSON` qui survient quand une valeur dynamique (`$fromAI`, `$json.…`, toute expression) est **injectée entre guillemets dans un JSON écrit à la main**. Après correction : la valeur est échappée automatiquement, quel que soit son contenu (guillemets, antislash, sauts de ligne).

## 2. Prérequis

1. Un node `<NODE_HTTP>` de type `httpRequestTool` ou `HTTP Request`, méthode POST, *Body Content Type* = JSON.
2. Le body contient au moins un champ `<PARAM>` dont la valeur = `$fromAI('<PARAM>', '<description>')` ou une expression.
3. Droit de **Save** sur le workflow `<WORKFLOW>`.

## 3. Procédure pas à pas

### 3.1 — Repérer l'anti-patron
1. Body de la forme (fragile) :
   ```
   ={ "<PARAM>": "{{ $fromAI('<PARAM>', '<description>') }}" }
   ```
   La valeur est placée entre guillemets sans échappement → tout `"` / `\` / retour ligne dans la valeur casse le JSON.

### 3.2 — Corriger (option A recommandée : JSON.stringify)
1. Ouvrir `<NODE_HTTP>` ; garder *Specify Body* = **Using JSON**.
2. Remplacer le contenu du champ JSON par (le champ étant en mode expression `fx`, **sans `=` initial**) :
   ```
   {{ JSON.stringify({ <PARAM>: $fromAI('<PARAM>', '<description>') }) }}
   ```
   Pour plusieurs champs : `{{ JSON.stringify({ <PARAM1>: $fromAI('<PARAM1>','…'), <PARAM2>: $fromAI('<PARAM2>','…') }) }}`.
3. **Save** le workflow.

### 3.3 — Alternative (option B : Using Fields Below)
1. *Specify Body* → **Using Fields Below**.
2. **Add Parameter** : *Name* = `<PARAM>` ; *Value* (mode expression) = `{{ $fromAI('<PARAM>', '<description>') }}`.
3. Vérifier que le paramètre nommé existe bien (sinon la clé JSON n'est pas reconstruite). **Save**.

## 4. Vérification / recette

1. Déclencher le workflow avec une entrée qui force des **caractères spéciaux** dans `<PARAM>` (guillemets + saut de ligne).
2. Attendu : `<NODE_HTTP>` en `success`, réponse HTTP normale. Confirmer sur une exécution réelle (le workflow actif ne prend la nouvelle version qu'après Save).

## 5. Difficultés rencontrées (typiques)

1. Passage en *Using Fields Below* mal terminé : wrapper retiré mais mode resté *Using JSON* → body = valeur brute non-JSON.
2. Édition non **sauvegardée** → l'exécution lit l'ancienne version (workflow actif).
3. **Double `=`** : coller `={{ … }}` dans un champ déjà en mode expression → `=={{ … }}` → `=` littéral en tête → JSON invalide.

## 6. Solutions implémentées

1. `JSON.stringify({...})` pour déléguer l'échappement (robuste, minimal).
2. **Save** puis re-test réel après chaque modif.
3. Contenu saisi **sans `=` initial** dans un champ déjà en mode expression.

## 7. Lessons learned

1. **Jamais de JSON écrit à la main** autour d'une valeur dynamique : sérialiser via `JSON.stringify` (ou *Using Fields Below*).
2. Champ n8n en mode expression `fx` : le contenu commence par `{{`, pas `={{`.
3. **Workflow actif ⇒ Save** avant tout re-test externe.
4. `$fromAI` reste détecté quel que soit l'emplacement → le schéma de l'outil (clés, descriptions) est préservé si on garde les mêmes noms.
5. Le même anti-patron se répète souvent sur **tous les agents partageant le même gabarit** → corriger en lot.

## 8. Références

- POS exact associée : `POS-LUMINA_Fix-Knowledge-JSON-body-Maestro_2026-07-09.md`.
- Erreur n8n de référence : `NodeOperationError: JSON parameter needs to be valid JSON` (HttpRequest V3).
