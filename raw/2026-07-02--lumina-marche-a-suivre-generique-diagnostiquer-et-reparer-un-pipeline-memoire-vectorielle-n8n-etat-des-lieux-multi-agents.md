---
type: raw
title: "LUMINA — Marche a suivre GENERIQUE — Diagnostiquer et reparer un pipeline memoire vectorielle n8n + etat des lieux multi-agents"
source_url: "drive:1wXESpHcI4V6VwEHvHbuVjD5UwGQHy8j-"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# Marche à suivre GÉNÉRIQUE — Diagnostiquer/réparer un pipeline de mémoire vectorielle (n8n + pgvector) & faire l'état des lieux d'un système multi-agents

**Portée :** patron réutilisable, indépendant de la marque. S'applique à tout système « RAG » sur n8n (embeddings OpenAI → pgvector) exposé via webhook + MCP, avec des agents hub-and-spoke.

> Convention : explications en français ; noms de nodes/credentials et code en anglais. Numérotation à partir de 1.

---

## A. Diagnostiquer un pipeline mémoire qui ne répond plus

### 1. Reproduire et qualifier l'erreur
- Appeler le point d'entrée (MCP / webhook) et noter le message **exact**.
- `NodeApiError` ⇒ un node interne plante. « Invalid JSON / empty » ⇒ le flux va au bout mais renvoie du vide (souvent 0 résultat).

### 2. Remonter au node fautif
1. n8n → **Executions**.
2. Filtrer les **Error** ; identifier le **workflow** réellement en échec (le serveur MCP n'est souvent qu'un relais qui « réussit »).
3. Ouvrir l'exécution → lire l'erreur du **node rouge**.

### 3. Suspects classiques (du plus fréquent au moins)
1. **Credential morte/rotée** (embeddings, DB) → ID orphelin.
2. **Table vide** (pipeline d'ingestion cassé, souvent TRUNCATE-puis-reload interrompu).
3. **SQL mal construit** (concaténation de chaînes avec contenu riche).
4. Seuil de similarité trop strict / dimension d'embedding incohérente.
5. Auth/permissions DB.

## B. Réparer

### 4. Credential morte
- Ouvrir le node concerné → re-sélectionner la credential dans le dropdown (ne pas se fier au nom affiché : le binding interne peut pointer un ID supprimé).
- **Balayer tous les workflows** qui utilisent la même credential (retrieval ET ingestion).

### 5. Table vide
- Vérifier l'historique : une ancienne exécution renvoyait-elle des lignes ? Si oui et que ça renvoie 0 maintenant → la table a été vidée.
- Repérer un node **TRUNCATE** en tête de l'ingestion : tout échec en aval (souvent l'embedding) laisse la table vide.
- Réparer la cause amont, puis **relancer l'ingestion** ; valider avec un `SELECT count(*)` (ou un node « Verify Table Count »).

### 6. SQL fragile → requêtes paramétrées (règle d'or)
Ne **jamais** construire de SQL par concaténation dans n8n quand le contenu est riche (markdown, apostrophes, backticks). Modèle :

```javascript
// Node "Build Insert" (Code)
const query =
  "INSERT INTO <table> (colA, colB, ..., embedding) " +
  "VALUES ($1, $2, ..., $N::vector) ON CONFLICT (<hash>) DO NOTHING;";
const values = [valA, valB, ..., '[' + emb.join(',') + ']'];  // vector = literal SANS quotes
return [{ json: { query, values } }];
```

```
// Node "Postgres" (Execute Query)
Query             = {{ $json.query }}
Options → Query Parameters (Expression) = {{ $json.values }}   // doit résoudre en Array
```

Pièges : ajouter l'option **Query Parameters** (sinon « there is no parameter $1 ») ; l'éditeur d'expression auto-ferme `{{ }}` (nettoyer les braces en trop) ; vérifier que le Result est bien un **Array**.

### 7. Valider end-to-end
Toujours re-tester par le **point d'entrée réel** (appel MCP/webhook), pas seulement au niveau du node.

## C. État des lieux d'un système multi-agents hub-and-spoke

### 8. Inventaire
- Rechercher par préfixe (ex. le préfixe de marque) ; lister les agents et leur **statut** (Published/Active).
- Distinguer les workflows **archivés/deprecated**.

### 9. Vérifier le câblage de l'orchestrateur (Superviseur)
- Trigger (chat / webhook) → node **AI Agent**.
- **Chat Model** (credential valide ? modèle ? API).
- **Memory** (historique conversationnel).
- **Tools** : outil mémoire (Knowledge) + un **Call-workflow** par spécialiste.

### 10. Vérifier chaque spécialiste
- Deux triggers : *chat* (test manuel) + *Executed by Another Workflow* (appel par le superviseur).
- AI Agent + Chat Model + (Chat Memory) + outil mémoire.
- Prompt utilisateur tolérant : `{{ $json.query || $json.chatInput }}`.
- **System message** : identité, périmètre/boundaries, « always query Knowledge first », langue de réponse, garde-fou de validation humaine, format de sortie.

### 11. Points de contrôle récurrents (review)
1. **Cohérence des noms d'outils** : le nom du node-outil doit matcher ce que le prompt demande d'utiliser.
2. **Modèle réellement configuré** vs documenté.
3. **Chat Memory en sous-workflow** : prévoir un fallback `sessionId` (sinon plantage quand appelé par un autre workflow).
4. **Langue & garde-fous** présents dans le prompt (validation humaine sur le critique).
5. **Qualité du contenu indexé** : la mémoire contient-elle vraiment ce que les agents doivent lire ?
6. **Sécurité** : auth du MCP (éviter « no auth » hors MVP) ; gating humain des actions critiques (coût, suppression, accès API).

## D. Challenges typiques & leçons

- **Erreur opaque** → toujours passer par Executions → node rouge.
- **Credential affichée OK mais binding mort** → réassigner explicitement ; balayer tous les workflows.
- **TRUNCATE-puis-reload** → un échec post-truncate vide la table (sauvegarde/idempotence à soigner).
- **SQL concaténé** → casse sur contenu riche ; passer aux **paramètres**.
- **Validation** → end-to-end, pas node par node.
- **Nommage** → séparer clairement *infra partagée* et *items de marque*.
