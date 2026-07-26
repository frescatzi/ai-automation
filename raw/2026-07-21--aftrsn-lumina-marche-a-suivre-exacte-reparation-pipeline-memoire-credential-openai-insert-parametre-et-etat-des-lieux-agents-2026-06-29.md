---
type: raw
title: "AFTRSN-LUMINA — Marche a suivre EXACTE — Reparation pipeline memoire (credential OpenAI + INSERT parametre) et etat des lieux agents — 2026-06-29"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — marche a suivre exacte — reparation pipeline memoire (credential openai + insert parametre) et etat des lieux agents — 2026-06-29.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# Marche à suivre EXACTE — Réparation du pipeline mémoire LUMINA + état des lieux des agents AFTRSN

**Date :** 2026-06-29
**Système :** n8n self-hosté (`n8n.aftersunpeople.com`) + mémoire centrale pgvector (`asp_memory`) + agents AFTRSN
**Validateur final :** Katel (Karter)
**Contexte :** la mémoire centrale (MCP `search_brand_memory` / `LUMINA-MCP-Server`) ne répondait plus. Objectif de la session : diagnostiquer, réparer, puis faire l'état des lieux des 4 agents.

> Convention : explications en français ; noms de workflows/nodes/credentials et code **en anglais** (comme dans n8n). Numérotation à partir de 1.

---

## 1. Symptôme de départ

Appel du MCP `search_brand_memory` depuis Claude → erreur `NodeApiError` (« The service was not able to process your request »). Après le 1ᵉʳ correctif, l'erreur change en « Invalid JSON in response body ». Conclusion : la chaîne mémoire est cassée quelque part dans n8n.

## 2. Localiser le node fautif (méthode)

1. Ouvrir n8n → **Overview** → onglet **Executions**.
2. Repérer les exécutions en **Error** (ici : `LUMINA-RETRIEVAL-MEMORY/WEBHOOK`, alors que `LUMINA-MCP-Server` renvoyait Success — il n'est qu'un relais).
3. Ouvrir l'exécution rouge → le node en rouge montre l'erreur exacte.

**Leçon :** dans n8n, ne jamais deviner — aller dans *Executions → exécution en échec → node rouge* pour lire l'erreur réelle.

## 3. Cause racine n°1 — credential OpenAI morte

Le node **`Embed Question (OpenAI)`** (HTTP Request POST `https://api.openai.com/v1/embeddings`) du workflow `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` affichait :

> `Credential with ID "DqFd9f5nr93sSXvU" does not exist for type "openAiApi".`

La credential OpenAI utilisée pour les embeddings avait été supprimée/remplacée (un nouvel ID a été créé le 24 juin sous le nom `OpenAi account`). Les nodes pointaient encore sur l'ancien ID mort.

### Correctif appliqué
1. Ouvrir le node `Embed Question (OpenAI)` → champ **OpenAi** (credential).
2. Même si le menu **affichait** « OpenAi account », re-sélectionner explicitement **OpenAi account** dans la liste (pour forcer un nouveau binding valide).
3. **Save** le workflow.

**Challenge :** le menu affichait le bon nom alors que le binding interne pointait vers l'ID mort.
**Solution :** re-sélectionner la credential dans le dropdown (ne pas se fier au nom affiché).
**Leçon :** une credential supprimée laisse les nodes avec un ID orphelin ; il faut réassigner explicitement.

## 4. Cause racine n°2 — la table `asp_memory` était VIDE

Après le correctif credential, l'exécution réussissait (3,5 s) mais `Postgres Search` renvoyait **0 ligne** → réponse vide → MCP « Invalid JSON ».

Vérification : une ancienne exécution réussie (20 juin) renvoyait **5 lignes** ; la même requête le 29 juin → **0 ligne**. Donc la table avait été **vidée** entre-temps.

Hypothèse confirmée : le workflow d'ingestion `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR` fait un **`TRUNCATE` en premier**, puis ré-embed + insert. Comme l'**embedding échouait** (même credential morte), la table était truncatée puis jamais re-remplie.

**Leçon :** quand une credential partagée meurt, balayer **tous** les workflows qui la référencent. Méfiance avec les pipelines « TRUNCATE puis reload » : un échec après le truncate laisse la table vide.

## 5. Cause racine n°3 — l'INSERT d'ingestion plantait (parsing SQL n8n)

Après avoir réparé la credential de l'ingestion (même méthode qu'au §3), le node **`Embed (OpenAI)`** réussissait (66 chunks embeddés), mais **`Postgres Insert`** échouait :

> `Syntax error at line 9 near "ai_os"`

Le node `Build Insert` construisait l'`INSERT` par **concaténation de chaînes**. Même avec les apostrophes échappées (`n''atteint`), le node Postgres de n8n **mal-parse** le contenu markdown (apostrophes + backticks) quand il découpe/analyse la requête. (n8n l'indique d'ailleurs sous le champ : « Consider using query parameters ».)

### Correctif appliqué — requêtes paramétrées

**Node `Build Insert` (Code, JavaScript)** — réécrit pour sortir une requête avec placeholders + un tableau `values` :

```javascript
const items  = $input.all();
const chunks = $('Chunk').all();
const out = [];
for (let i = 0; i < items.length; i++) {
  const d = items[i].json.data;
  const emb = d && d[0] && d[0].embedding;
  if (!emb) { continue; }
  const m = chunks[i].json;
  const vec = '[' + emb.join(',') + ']';            // pgvector literal, NO quotes
  const query =
    "INSERT INTO asp_memory " +
    "(title, content, collection, knowledge_type, source, source_ref, content_hash, embedding) " +
    "VALUES ($1, $2, $3, $4, $5, $6, md5($2), $7::vector) " +
    "ON CONFLICT (content_hash) DO NOTHING;";
  const values = [
    m.title ?? '', m.content ?? '', m.collection ?? '',
    m.knowledge_type ?? '', m.source ?? '', m.source_ref ?? '', vec,
  ];
  out.push({ json: { query, values } });
}
return out;
```

**Node `Postgres Insert`** :
1. Operation = **Execute Query** ; Query = `{{ $json.query }}`.
2. **Add option → Query Parameters**.
3. Basculer le champ en **Expression** et saisir `{{ $json.values }}` (doit résoudre en **Array**).

**Challenge A :** `Query Parameters` non ajouté au début → erreur « there is no parameter $1 ».
**Solution A :** ajouter l'option et la mettre en mode Expression renvoyant le tableau `values`.

**Challenge B :** l'éditeur d'expression n8n auto-ferme `{{ }}` → on obtient `{{ $json.values }} }}` (braces en trop).
**Solution B :** supprimer les 3 caractères en trop ; vérifier que le **Result** affiche `[Array: [...]]`.

**Leçon :** ne jamais construire de SQL par concaténation dans n8n ; utiliser des **paramètres** (`$1..$n` + tableau via expression). C'est plus robuste et plus sûr (anti-injection).

## 6. Repeuplement et vérification

1. **Save** le workflow d'ingestion.
2. **Execute workflow** (déclencheur manuel) → tous les nodes verts.
3. Node **`Verify Table Count`** (`SELECT source_ref, count(*) ... GROUP BY source_ref`) → **25 fichiers / 66 chunks**.
4. **Publish** le workflow avec une note de version.
5. **Test final end-to-end :** appel du MCP `search_brand_memory` → renvoie bien des résultats avec scores de similarité. ✅

**Leçon :** toujours valider de **bout en bout** (l'appel MCP réel), pas seulement au niveau d'un node.

## 7. État des lieux des 4 agents AFTRSN

Recherche `AFTRSN` dans n8n → 4 agents (+ 1 archivé `ZZZ_…-DEPRECATED` à ignorer), tous **Published** :

| Agent | Dossier | Rôle |
|---|---|---|
| **AFTRSN-SUPERVISOR** | AFTRSN-AGENTS | Orchestrateur, point d'entrée |
| **AFTRSN-BRAND-GUARDIAN** | Sub-Agents | Cohérence & voix de marque |
| **AFTRSN-CHANNEL-CONTENT** | Sub-Agents | Contenu & canaux |
| **AFTRSN-ACQUISITION-PERFORMANCE** | Sub-Agents | Campagnes, KPIs, acquisition |

### Superviseur — câblage réel
- Trigger **When chat message received** → node **AI Agent**.
- **Chat Model** : OpenAI Chat Model — credential **OpenAi account** (valide), modèle **gpt-4.1-mini**, Responses API ON.
- **Memory** : Chat Memory (Postgres).
- **Tools** : **Knowledge Bank** (→ webhook `memory-search`) + **Call AFTRSN-Brand-Guardian** + **Call AFTRSN-Channel-Content** + **Call AFTRSN-Acquisition-Performance**.

### Spécialiste (Brand-Guardian, représentatif des 3)
- **Deux** triggers : *When chat message received* (test manuel) ET *When Executed by Another Workflow* (appel par le Superviseur).
- AI Agent → Chat Model (OpenAI) + Chat Memory (Postgres) + **Knowledge Bank** (tool).
- Prompt utilisateur : `{{ $json.query || $json.chatInput }}`.
- **System message** (résumé) : identité (« You are BRAND-GUARDIAN… of AFTER SUN PEOPLE (AFTRSN) ») ; règle de nommage (AFTER SUN PEOPLE/AFTRSN = marque, « After Sun » = l'événement) ; **« Before answering, ALWAYS query the central memory with the "Knowledge" tool »** ; format de sortie (verdict on-brand / needs changes / off-brand, déviations + règle exacte enfreinte, réécritures avant/après) ; « precise and constructive, never vague ».

### Décision de nommage (tranchée cette session)
Les 4 agents restent **`AFTRSN-`** car spécifiques à la marque ASP. L'infra partagée (MCP, workflows mémoire, ingestion, pgvector) reste **`LUMINA-`**. Le MCP a justement été renommé **`LUMINA-MCP-Server`** (infra).

## 8. Constats de révision (à traiter ensuite)

1. **Nom de l'outil mémoire incohérent** : le node s'appelle **`Knowledge Bank`** mais le system prompt dit d'utiliser l'outil **« Knowledge »** → aligner (renommer le node en `Knowledge` ou corriger le prompt).
2. **Modèle** : réel = **gpt-4.1-mini** (la doc/passation disait gpt-4o-mini) → corriger la doc.
3. **System prompt des spécialistes** : ne contient pas explicitement « respond in the user's language » ni le garde-fou « Katel valide le critique » que la passation décrivait → à ajouter si voulu.
4. **Chat Memory dans les spécialistes** : vérifier le fallback `sessionId` quand ils sont appelés en sous-workflow (sinon risque de plantage), comme noté dans la passation.
5. **Contenu de la mémoire** : `asp_memory` contient surtout le wiki système (docs LUMINA), peu de contenu de **marque** → le Brand-Guardian est sous-alimenté tant que la bible n'entre pas dans le pipeline indexé.
6. **Sécurité MCP** : auth = aucune (MVP) → durcir (OAuth2) ; marquer Katel-gated les tâches critiques.

## 9. Résumé des leçons apprises

- Diagnostiquer via *Executions → node rouge*, pas à l'aveugle.
- Une credential supprimée casse silencieusement **tous** les workflows qui la référencent ; réassigner explicitement.
- Attention aux pipelines **TRUNCATE-puis-reload** : un échec après le truncate vide la table.
- **Jamais** de SQL par concaténation dans n8n → **requêtes paramétrées**.
- L'éditeur d'expression n8n auto-ferme `{{ }}` (braces en trop à nettoyer).
- Valider **end-to-end** (appel MCP réel).
- Convention de nommage : **AFTRSN = marque**, **LUMINA = infra partagée**.
