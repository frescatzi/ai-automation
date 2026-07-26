---
type: wiki
title: "AFTRSN — Méthode de construction d'un workflow n8n"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [aftrsn/raw/2026-07-21--aftrsn-marcheasuivre-construction-n8n.md]
related: [[architecture-processus-metier]]
updated: 2026-07-21
---

# AFTRSN — Méthode de construction d'un workflow n8n

Référence opérationnelle pour construire ou modifier un workflow n8n AFTRSN via automatisation navigateur (Claude in Chrome). Rédigée après la construction de `AFTRSN-Event-Creator` (P1, voir [[architecture-processus-metier]]) et enrichie sur les workflows suivants (Venue-Coordinator, Budget-Tracker, brief artistes J-7).

## Avant de commencer

- Lire en entier la spec du workflow (nodes, types n8n, contrat d'entrée `jsonExample`).
- **n8n live fait foi sur la spec en cas de désaccord** — toujours vérifier le workflow réel avant d'agir (ex. vécu : la spec prévoyait un node "Respond to Webhook" en sortie, alors que le trigger réel était `When Executed by Another Workflow`, incompatible avec ce type de node).
- Vérifier le schéma réel des bases Notion ("The Backstage") plutôt que de se fier à un schéma théorique d'une ancienne passation.
- Repérer et réutiliser les sous-workflows Lumina OS existants (`LUMINA-MEMORY-WRITE/WEBHOOK`, `LUMINA-AI-Router`…) via le MCP n8n (`search_workflows`, `get_workflow_details`) plutôt que d'inventer un endpoint. Pour connaître le contrat exact d'un sous-workflow appelé en `Execute Workflow`, ouvrir son node trigger dans l'éditeur : le "JSON Example" liste les champs attendus.

## Conventions AFTRSN

- **Vocabulaire Notion :** "Edition" (jamais "Event"). DB réelles : Editions, Venues, DJs/Artists, Budget & Finance, Content & Social, Marketing & KPI, Partners & Sponsors, Media Assets, Cities. Pas de DB "Event Tasks" séparée — la checklist vit dans le corps de la page Edition.
- **Nommage dossiers Drive :** `YYYYMMDD_AFTRSN_Photos` / `YYYYMMDD_AFTRSN_Videos`, chacun avec sous-dossiers `Raw` et `Edited`. Format strict, sans dérogation.
- **Comptes Google :** AFTRSN a deux comptes (Hello = B2C, Partners = B2B) ; Drive/Calendar côté P1 utilise le compte Hello.
- **Mémoire épisodique :** tout run significatif écrit un épisode dans `aftrsn_memory` (collection `episodic`) via `LUMINA-MEMORY-WRITE/WEBHOOK`, appelé en **Execute Workflow** (jamais HTTP Request) avec le contrat exact `{brand:"aftrsn", title, content, collection, knowledge_type, source, source_ref}` — le trigger est en mode `jsonExample` : reproduire le contrat exactement (champ manquant ou en trop = ignoré silencieusement).
- **Modèles Claude :** jamais le node natif Anthropic (bug 404 connu) — toujours un node HTTP Request.
- **Secrets :** Claude ne saisit jamais de credential (Client ID/Secret, mot de passe, API key) — il guide l'utilisateur pour qu'il les saisisse lui-même dans l'UI n8n.

## Construire dans le canvas

1. Construire node par node, dans l'ordre de la spec, en connectant chaque nouveau node via le "+" du connecteur.
2. Pour agréger deux branches parallèles, insérer un node **Merge** (Append, 2 inputs) comme simple point de synchronisation — sans lui, n8n exécute le node suivant séparément par branche. Les nodes en aval continuent de lire les données d'origine via `$('NomDuNode').item.json...`, pas via le contenu du Merge.
3. Renommer chaque node dès sa configuration terminée (jamais garder "HTTP Request", "Create a database page2"…) — sinon le canvas devient illisible.
4. **Ne jamais cliquer "Execute step" sur un node dont l'amont n'a pas de données en cache** : ça relance toute la chaîne depuis le trigger avec des données de test nulles, et peut créer de vraies données parasites (dossiers Drive `_` fantômes constatés plusieurs fois). Pour un test isolé, utiliser "set mock data".
5. Pour valider un long JSON injecté (checklist Notion…), remonter visuellement vérifier le début plutôt que de se fier seul à l'indicateur de validité de l'éditeur.

## Pièges d'expressions et d'UI n8n

- Ne jamais taper le `=` de tête dans un champ Expression/resourceLocator — le mode Expression l'implique déjà (taper seulement `{{ ... }}`), sinon un `=` en trop casse les lookups (ex. Drive).
- Un champ qui affiche `{{...}}` en brut au lieu d'une valeur calculée est resté en mode "Fixed" — basculer en "Expression".
- L'éditeur CodeMirror auto-ferme les accolades : ne jamais retaper `}}`, sinon il se duplique.
- Dans un node Set/Edit Fields, si le champ "value" reste vide après avoir tapé "name", cliquer ailleurs puis re-cliquer sur "value" avant de taper (le focus ne prend pas toujours au premier clic).
- Un panneau de recherche de node tronqué en bord d'écran reste cliquable tel quel, ou se corrige en fermant/rouvrant.
- Pour le dropdown "From list"/"By ID" d'Execute Workflow, préférer les flèches clavier (`Down` + `Return`) au clic, trop imprécis sur un menu qui se referme vite. Utiliser systématiquement **By ID** (pas la liste) quand deux DB portent le même nom (ancien/nouveau Backstage) — la liste ne montre pas l'arborescence.
- Auditer les champs expression suspects en vérifiant qu'ils commencent par `=` et ferment bien leurs `}}` : le symptôme runtime d'une expression malformée est du texte brut (`{{ $('Node')... }}`) dans l'output.

## Exécution, données et sorties multi-items

- Un node de recherche à 0 item ne déclenche pas ses successeurs, sans erreur visible. Activer **Always Output Data sur le node de recherche lui-même** (pas le suivant), et tester la présence d'un champ (`i.json.id`) plutôt que `items.length` — un AOD produit un item vide qui compte pour 1.
- La sortie d'un sous-workflow est la sortie de son dernier node : si celui-ci tourne par item (ex. Set après un Merge multi-branches), le mapping en aval produit des `undefined`. Mettre **executeOnce** sur le node de sortie de chaque sous-workflow (contrat : 1 item en sortie) et référencer en `.first()` plutôt que `.item` côté orchestrateur.
- Pour un sous-workflow appelé sur N items, utiliser **`mode: 'each'`** (Execute Workflow v1.3) pour obtenir une exécution isolée par item plutôt qu'un mode "once with all items" qui mélange les données.
- Toujours inspecter la vraie sortie d'une exécution avant d'écrire un parseur (ex. le node Gmail simplify-off renvoie déjà `text`, `html`, `subject`, `from.text`… pas de payload brut ; le node Notion getAll simplifié renvoie des clés `property_*` en snake_case, pas les noms de propriétés Notion).
- Pour inspecter une exécution volumineuse sans geler l'onglet (le détail `includeData=true` peut contenir des traces d'agent énormes), préférer vérifier par les effets (relire Notion/Gmail/Telegram) ou des tests booléens ciblés sur `resultData.runData` plutôt qu'un décodage récursif naïf du format "flatted" — une vérification `raw.includes('"NomDuNode"')` peut faussement matcher le `workflowData` complet embarqué dans le payload.

## Refactoring et API REST interne n8n

Le refactoring lourd (découpage en sous-workflows) est très lent via le canvas. Alternative fiable : l'API REST interne, appelée depuis la console de l'onglet connecté avec le header `browser-id` (`localStorage.getItem('n8n-browserId')`) :
- `GET/POST/PATCH /rest/workflows/...` pour lire/créer/modifier un workflow entier (nodes + connections + versionId) en masse — toujours vérifier ensuite par rechargement de l'éditeur.
- **Recharger l'éditeur immédiatement après tout PATCH** : un onglet resté ouvert autosave son état périmé et peut écraser silencieusement le PATCH.
- Jamais de mutation directe du store Pinia : n8n ne la détecte pas, cmd+s devient un no-op.
- `POST /rest/workflows/<id>/run` pour exécuter un workflow sans passer par les boutons UI (fragiles : références DOM périmées, bouton déplacé par le panneau Logs) ; vérifier qu'un **nouveau** run apparaît dans `/rest/executions?limit=1` plutôt que d'attendre à l'aveugle.
- `POST /rest/workflows/<id>/activate` avec `{versionId, name, description}` (versionId = celui du GET courant) pour publier par API.
- Pour tester un sous-workflow isolément avec un payload contrôlé, construire un appelant jetable (Manual Trigger → Execute Workflow), re-PATCHé à chaque cas de test.

## Publication

- Un orchestrateur ne peut pas être publié avant ses sous-workflows — les publier d'abord, l'orchestrateur en dernier. Un callee doit être publié même pour un simple test manuel de l'appelant.
- Un PATCH API après publication crée une **version non publiée** : re-publier après toute modification post-publication (constaté sur un changement de timezone : cron "9h" décalé à 11h suisses par défaut UTC → fixer `settings.timezone = 'Europe/Zurich'` puis republier).
- Une erreur `409 conflict with one of the webhooks` sur un workflow dupliqué vient d'un `webhookId` hérité — régénérer l'UUID avant d'activer.
- Pour étendre un workflow déjà publié et testé sans re-tester les branches existantes, préférer une **chaîne parallèle additive** branchée sur le même trigger plutôt que de modifier le flux existant.

## Intégrations externes — pièges observés

- **Google Sheets (batchGet) :** n8n sérialise les query params du node HTTP en objet — des noms de paramètres dupliqués (plusieurs `ranges`) s'écrasent silencieusement. Passer les paramètres répétés inline dans l'URL (`?ranges=…&ranges=…` encodé), jamais via `queryParameters`.
- **API Google (Gmail, Sheets) en 403 :** l'API n'est pas activée dans le projet Google Cloud du client OAuth — l'activer dans la console (`console.developers.google.com/apis/api/<service>.googleapis.com`), ~2 min de propagation. Sur cette instance la console est protégée par passkey : activation à faire faire par Karter, vérification par l'effet ensuite.
- **Notion "Error fetching options" / autosave "Unauthorized" en boucle :** session n8n expirée, pas une perte de configuration — se reconnecter d'abord. Les clés de propriété par nom survivent à un changement de DB si les schémas coïncident.
- **Telegram — chat_id introuvable :** l'API bots ne résout pas les `@username` en privé. Solution la plus rapide : l'utilisateur envoie un message à `@rawdatabot`, qui renvoie son `chat.id`.
- **Credentials illisibles depuis l'outil JS navigateur** (`/rest/credentials` bloqué "sensitive") : lire les ids dans les nodes de workflows existants plutôt que par l'API.
- **`jsCode` d'un node existant illisible** (même filtre "sensitive", faux positif sur du code) : reconstruire le contrat depuis les autres nodes lisibles (Switch, Gmail, Telegram) et les runbooks de passation ; en dernier recours, lire dans l'éditeur UI.
- **Donnée de test "active" reprise par un process planifié :** neutraliser la fiche dans un statut que le process ignore (ex. `Cancelled`) plutôt que la supprimer soi-même — suppression définitive réservée à Karter.

## Definition of Done d'un workflow AFTRSN

- Tous les nodes de la spec présents, connectés, renommés clairement.
- Aucun credential saisi par Claude — uniquement configuré par Karter avec guidage.
- Le workflow écrit un épisode mémoire (`aftrsn_memory`, collection `episodic`) à la fin d'un run réussi.
- Aucune écriture partielle silencieuse sur la branche d'erreur.
- Testé au moins une fois de bout en bout avec un payload réaliste (pas seulement vérifié visuellement).
- Publié ("Publish") dans n8n une fois validé.

## Règle de construction transverse

Vue simple et limitée par étape : un gros processus se découpe en sous-workflows métier appelés par un orchestrateur, chaque sous-workflow ne recevant que ses propres paramètres — jamais de référence croisée `$('Node')` entre blocs (voir [[architecture-processus-metier]] §Règles de construction).
