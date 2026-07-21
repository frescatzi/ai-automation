---
type: wiki
title: "SOP — Réparer une credential Postgres partagée fantôme (n8n / mémoire agents)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-07-21--pos-lumina-reparation-credential-postgres-partagee-20260706.md
related:
  - concept-n8n-credentials
  - sop/sop-audit-edition-n8n-api-interne
  - sop/sop-diagnostiquer-pipeline-memoire-vectorielle
  - sop/SOP_installer-pgvector-sur-postgres-coolify
updated: 2026-07-21
---

# SOP — Réparer une credential Postgres partagée fantôme (n8n / mémoire agents)

**Contexte :** une seule credential Postgres est souvent câblée dans des dizaines de workflows n8n (écriture/lecture mémoire, ingestion, Chat Memory de tous les agents). Si elle se corrompt, **tout le « cerveau »** (mémoire + agents type Maestro/Hermès) tombe en même temps — point unique de défaillance à traiter comme un incident critique.

## 1. Symptôme — credential fantôme

Erreur sur tout node Postgres (`n8n-nodes-base.postgres`, `memoryPostgresChat`) :
```
Credential with ID "<id>" does not exist for type "postgres".
```
La credential **apparaît dans la liste** (`GET /rest/credentials`) mais `GET /rest/credentials/<id>` renvoie **404** — métadonnées listées, blob chiffré illisible. Un redémarrage Docker ne répare rien : le défaut est dans la donnée, pas dans le process.

## 2. Mesurer le rayon de la panne avant d'agir

Scanner tous les workflows pour lister ceux qui référencent la credential morte (actifs vs inactifs) :
```js
const OLD='<id>';
const list=(await (await fetch('/rest/workflows?limit=250',{headers})).json()).data;
for(const wf of list){ const w=(await (await fetch('/rest/workflows/'+wf.id,{headers})).json()).data;
  const hit=w.nodes.filter(n=>n.credentials?.postgres?.id===OLD).map(n=>n.name);
  if(hit.length) console.log(w.active?'ACTIF':'inactif', w.name, hit); }
```

## 3. Diagnostic — ce n'est pas la base de données

L'erreur (`functionality:'configuration-node'`) est levée **au chargement de la credential**, avant toute requête SQL : les données pgvector (`aftrsn_memory`, `lumina_memory`, `n8n_chat_histories`) sont intactes. Seul l'objet credential n8n doit être recréé.

## 4. Retrouver les paramètres de connexion

La credential fantôme étant illisible, retrouver host/port/db/user ailleurs (stack Coolify/Dokploy « n8n + postgres ») :
- Terminal du conteneur **Postgres** → `env` : `COOLIFY_CONTAINER_NAME=postgresql-<hash>` = **host interne**, `POSTGRES_DB`, `POSTGRES_USER`.
- Vérifier où sont réellement les tables (souvent pas dans la base `postgres` mais dans `n8n`) :
  ```
  psql -U <user> -d n8n -c "\dt" | grep -iE 'aftrsn_memory|lumina_memory|chat_histories'
  ```
- Host = **nom du conteneur** (jamais `localhost`, n8n et Postgres sont deux conteneurs distincts). Port 5432, SSL off (réseau interne).

## 5. Recréer la credential — action HUMAINE uniquement

L'utilisateur crée la nouvelle credential Postgres dans l'UI n8n (host = conteneur, port 5432, database = celle des tables, user + **mot de passe saisis par lui**). Test connection → Save → nouvel `id`. **L'agent ne saisit jamais de secret.**

## 6. Repointer tous les nodes — action agent (sans secret)

Changer un `id` de credential dans un node est de la config, pas un secret → l'agent le fait en masse via l'API interne (cf. [[sop/sop-audit-edition-n8n-api-interne]]) :
```js
const OLD='<ghost>', NEW='<newId>', NAME='<newName>';
for(const wf of list){ const w=(await (await fetch('/rest/workflows/'+wf.id,{headers})).json()).data;
  let n=0; for(const node of w.nodes){ if(node.credentials?.postgres?.id===OLD){ node.credentials.postgres={id:NEW,name:NAME}; n++; } }
  if(n) await fetch('/rest/workflows/'+wf.id,{method:'PATCH',headers,body:JSON.stringify({name:w.name,nodes:w.nodes,connections:w.connections,settings:w.settings})}); }
```

## 7. ⚠️ Piège central — republier les workflows actifs

**Un `PATCH` de credential met à jour le brouillon, pas la version publiée/active.** Les **webhooks et déclencheurs** actifs continuent d'exécuter l'ancienne credential tant qu'on n'a pas republié — alors qu'`executeWorkflow` lit lui le brouillon frais. D'où le symptôme piège : l'écriture (via executeWorkflow) semble marcher, la lecture par webhook reste cassée.

Republier chaque workflow actif repointé (même mécanique que [[sop/sop-audit-edition-n8n-api-interne]]) :
```js
const w=(await (await fetch('/rest/workflows/'+id,{headers})).json()).data;
await fetch('/rest/workflows/'+id+'/activate',{method:'POST',headers,
  body:JSON.stringify({versionId:w.versionId,name:w.name,description:w.description||''})});
```
Un simple toggle actif→off→on ne suffit **pas** ; seul `/activate` avec le `versionId` courant promeut le brouillon.

## 8. Vérifier la reprise

1. **Écriture** : appeler LUMINA-MEMORY-WRITE (episodic) via un appelant jetable.
2. **Lecture** : `POST /webhook/memory-search {question}` → doit renvoyer 200 + contenu.
3. **Agent** : exécuter Maestro en chat → doit charger sa Chat Memory, faire une recherche mémoire et répondre.

## 9. Durcissement obligatoire après incident

Construire un health-check mémoire planifié (`LUMINA-HEALTHCHECK-MEMOIRE`) : Schedule quotidien → `POST /webhook/memory-search` (neverError + fullResponse) → Code évalue `statusCode===200` → IF → alerte Telegram si KO, silence si OK. Évite de redécouvrir une panne totale par hasard.

## Leçons clés

- **Brouillon ≠ version publiée** : après tout changement de credential/paramètre sur un workflow actif, republier via `/activate` — sinon les triggers gardent l'ancienne config.
- **`executeWorkflow` lit le brouillon frais ; webhook/trigger lit le publié** : un symptôme « écriture OK / lecture KO » pointe vers ce décalage.
- **Une credential partagée = point unique de défaillance** — à monitorer activement, pas seulement à réparer après coup.
- `configuration-node` « credential does not exist » = toujours un problème d'objet credential n8n, jamais de la base ni des données.

## Voir aussi

- [[concept-n8n-credentials]] — gestion générale des credentials n8n (types, bonnes pratiques).
- [[sop/sop-audit-edition-n8n-api-interne]] — mécanique API interne PATCH/activate utilisée ici pour le repoint en masse.
- [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] — diagnostic générique d'un pipeline RAG n8n+pgvector (la « credential morte » y est un suspect classique ; cette page détaille le cas limite credential *partagée fantôme*).
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — setup de la base vectorielle concernée.
