---
type: raw
title: "POS-GENERIQUE_Etat-conversation-clarification-bot-n8n"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_etat-conversation-clarification-bot-n8n.md"
captured: 2026-07-21
vault: brands
brand: n8n
immutable: true
---

# POS GÉNÉRIQUE — Rendre une clarification « sans état » résolutive dans un bot n8n (état de conversation)

Patron réutilisable pour tout bot conversationnel n8n (Telegram, WhatsApp, chat web…) dont la passerelle **pose une question de précision puis oublie tout** : la réponse de l'utilisateur est reclassée comme un message neuf → boucle. On ajoute un **état de conversation persistant** minimal.

## 1. Symptôme & cause racine
- Symptôme : répondre à une question de clarification (« quelle base/marque/date ? ») ne résout jamais ; ça reboucle.
- Cause : la passerelle est **sans état**. À l'émission de la question de clarification, l'exécution se termine ; rien ne mémorise la question d'origine, donc la réponse suivante n'est pas fusionnée avec elle.

## 2. Principe du correctif
Une table clé/valeur par utilisateur : `conversation_state (chat_id PK, original_question, created_at)`. Trois greffes sur le flux :
1. **Lire + purger** à l'entrée (avant le classifieur) : si un état est en attente pour ce `chat_id`, on le récupère et on l'efface.
2. **Fusionner** : si état présent, `message_effectif = question_origine + " — précision: " + réponse_utilisateur`, puis on laisse le classifieur re-router avec le contexte complet.
3. **Écrire** à l'émission de la clarification : on enregistre la question d'origine (idéalement déjà fusionnée, pour gérer les clarifications enchaînées).

## 3. Schéma de table
```sql
CREATE TABLE IF NOT EXISTS conversation_state (
  chat_id text PRIMARY KEY,
  original_question text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```
Créer la table **une fois** (étape séparée) — ne pas la recréer à chaque message via multi-statement (voir §6).

## 4. Node « Lire + purger état » (sur le chemin d'entrée)
```sql
WITH del AS (
  DELETE FROM conversation_state
  WHERE chat_id = $1 AND created_at > now() - interval '10 minutes'
  RETURNING original_question
)
SELECT (SELECT original_question FROM del) AS pending_question;
```
- Un **seul statement** (CTE) → compatible paramètre `$1`.
- **Toujours 1 ligne** grâce au scalar-subquery (`pending_question = null` si rien) → ne casse jamais le flux principal.
- **Lit et purge** atomiquement ; **expiration** (timeout) intégrée par `created_at`.
- `queryReplacement` en **tableau** : `={{ [<expr chat_id> ] }}`.

## 5. Fusion (node Set/Router déjà présent)
Rendre le champ de message conditionnel :
```
{{ <read>.pending_question
   ? (<read>.pending_question + " — précision: " + <message courant>)
   : <message courant> }}
```
Astuce robustesse : si les nodes en aval lisent le message via une **référence nommée** (`$('<node parse>').item.json.<champ>`) et non `$json`, on peut insérer le node de lecture sans casser la chaîne.

## 6. Node « Écrire état » (branche de clarification)
```sql
INSERT INTO conversation_state (chat_id, original_question, created_at)
VALUES ($1, $2, now())
ON CONFLICT (chat_id) DO UPDATE SET original_question = EXCLUDED.original_question, created_at = now();
```
- `queryReplacement` en **tableau** : `={{ [<expr chat_id>, <expr question fusionnée> ] }}`.
- Stocker la question **fusionnée** (sortie du node de préparation) plutôt que le message brut → gère les clarifications en cascade.

## 7. Règles d'or (n8n Postgres)
1. **Chemin critique** = tout node dont l'échec bloque l'entrée : garantir **1 ligne** en sortie (scalar-subquery), sinon 0 item casse le flux.
2. **1 statement par node** : le multi-statement (`;`) est incompatible avec les paramètres `$1` (protocole étendu). Créer la table à part, une fois.
3. **Droits** : table de contrôle dans une base/credential **en écriture** (pas un compte lecture-seule type ACL). Symptôme d'erreur de droits : `permission denied for schema public`.
4. **`queryReplacement` en tableau** pour tout texte utilisateur (virgules/apostrophes sûres) ; jamais l'interpolation brute de texte dans le SQL.
5. **Valider sur un workflow scratch** (Manual → nodes PG) avant d'exposer au trafic ; exécuter via le bouton UI, lire le résultat.

## 8. Test d'acceptation
- Message ambigu → le bot pose la question de clarification ET un état est écrit (vérifier la branche `… → question → écrire état`).
- Réponse de précision → le node de lecture renvoie la question d'origine, fusion, le classifieur route correctement (plus de clarification) → réponse réelle.
- Après T minutes sans réponse, l'état expire (le `WHERE created_at > now() - interval 'T'` l'ignore).

## 9. Pièges
- Node PG à 0 ligne → 0 item (fuite du flux) : toujours le scalar-subquery.
- Éditer une passerelle hors scope du connecteur : passer par l'API interne `/rest/*` (`GET` versionId → `PATCH` → `POST /activate`).
- Variables `window` non partagées entre onglets ; suppression d'un workflow = `archive` puis `DELETE`.

---
*POS GÉNÉRIQUE — 2026-07-12. Version exacte : `POS-EXACT_Etat-conversation-clarification-Gateway_2026-07-12.md`. S'applique à toute passerelle de bot n8n récente avec classifieur d'intention + branche de clarification.*
