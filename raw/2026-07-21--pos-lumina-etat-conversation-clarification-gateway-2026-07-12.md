---
type: raw
title: "POS-LUMINA_Etat-conversation-clarification-Gateway_2026-07-12"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-lumina_etat-conversation-clarification-gateway_2026-07-12.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS LUMINA — État de conversation du Gateway Telegram (clarification résolutive) — 2026-07-12

**Scope :** LUMINA OS / Bot Lyra — `LUMINA-TELEGRAM-GATEWAY` (`uGF3pSv6aTK79cqi`).
**Statut :** en service (version `959ef7c8`, active), validé live (exéc. #5291 + #5293).

## But
Garantir qu'une question de clarification du bot (« quelle marque ? ») aboutisse : la réponse de l'utilisateur est fusionnée avec la question d'origine et re-routée, au lieu de reboucler.

## Préconditions
1. Accès n8n (`n8n.aftersunpeople.com`) connecté ; édition par l'API interne `/rest/*` (Gateway hors scope du connecteur).
2. Credential Postgres **applicative `LUMINA_Postgres`** (`mlN3noHH3TT9C6Eu`) disponible sur l'instance.
3. Table `conversation_state` créée dans cette base (voir étape 1).

## Procédure (numérotée à partir de 1)
1. **Créer la table** `conversation_state (chat_id text PRIMARY KEY, original_question text NOT NULL, created_at timestamptz NOT NULL DEFAULT now())` dans `LUMINA_Postgres` (étape unique ; ne pas la recréer par message).
2. **Insérer le node `Lire+purger état`** (Postgres `executeQuery`, cred `LUMINA_Postgres`) entre `Is it /help?`(false) et `Prepare Router input`. Requête = CTE `WITH del AS (DELETE … WHERE chat_id=$1 AND created_at > now() - interval '10 minutes' RETURNING original_question) SELECT (SELECT original_question FROM del) AS pending_question;` ; `queryReplacement = {{ [$('Parse the message').item.json.chat_id] }}`.
3. **Modifier l'assignment `input`** de `Prepare Router input` : fusion conditionnelle `pending_question ? (pending_question + " — précision: " + input_courant) : input_courant`.
4. **Insérer le node `Écrire état`** (Postgres `executeQuery`, cred `LUMINA_Postgres`) après `Ask which brand`. Requête = `INSERT … VALUES ($1,$2,now()) ON CONFLICT (chat_id) DO UPDATE SET original_question=EXCLUDED.original_question, created_at=now();` ; `queryReplacement = {{ [$('Parse the message').item.json.chat_id, $('Prepare Router input').item.json.input] }}`.
5. **Câbler** : `Is it /help?`[false] → `Lire+purger état` → `Prepare Router input` ; `Ask which brand` → `Écrire état`.
6. **Publier** : `PATCH /rest/workflows/<id>` puis `POST /rest/workflows/<id>/activate` (versionId renvoyé par le PATCH).

## Vérification
- `GET` de contrôle : version à jour, `active:true`, les 2 nodes sur `LUMINA_Postgres`, node de lecture en statement unique, fusion présente, connexions correctes.
- Test Telegram : (a) message ambigu → question « quelle marque ? » + branche `Écrire état` ; (b) réponse marque → `Lire+purger état` restitue la question, fusion, `Question for Memory-Search ?` → réponse mémoire réelle (plus de boucle).

## Rollback
Retirer les 2 nodes, reconnecter `Is it /help?`[false] → `Prepare Router input`, restaurer `input = {{ $('Parse the message').item.json.input }}`, puis `activate`. Table conservable (inerte).

## Invariants (à ne pas violer)
- Table de contrôle dans `LUMINA_Postgres` (écriture) — **jamais** `LUMINA_UserAccess_Postgres` (lecture ACL, `permission denied`).
- Node de lecture sur **chemin critique** → doit toujours renvoyer **1 ligne** (scalar-subquery).
- **1 statement paramétré** par node ; `queryReplacement` en **tableau** pour le texte utilisateur.

## Difficultés rencontrées
- Gateway hors scope connecteur → API interne obligatoire.
- Credential ACL en lecture seule → `permission denied for schema public`.
- Node PG à 0 ligne = 0 item → risque de casser le flux d'entrée.
- Multi-statement incompatible avec les paramètres `$1`.
- Texte utilisateur (virgules/apostrophes) dans les paramètres.

## Solutions implémentées
- Base `LUMINA_Postgres` pour l'écriture ; table créée une fois.
- Scalar-subquery + CTE `DELETE … RETURNING` → lecture+purge atomique, toujours 1 ligne, expiration 10 min.
- `queryReplacement` en tableau (validé avec valeur piégée virgule+apostrophe).
- Validation complète sur workflow scratch avant exposition au trafic.

## Lessons learned
- Optimiser/valider le **chemin d'entrée** avec la même rigueur que l'orchestrateur : un node d'entrée cassé rend le bot muet.
- Le bon pattern SQL (scalar-subquery + CTE lire/purger) évite à la fois la boucle et les 0-item.
- API interne n8n = levier fiable pour éditer les workflows hors scope du connecteur ; toujours `activate` après `PATCH`.

---
*POS LUMINA — 2026-07-12, Claude (Cowork). Réf. : Bible 360 v2.2, SPEC-BUILD Gateway/Router. Exact : POS-EXACT_… ; Générique : POS-GENERIQUE_Etat-conversation-clarification-bot-n8n.md.*
