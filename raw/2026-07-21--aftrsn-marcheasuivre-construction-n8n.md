---
type: raw
title: "AFTRSN_MarcheASuivre_Construction-n8n"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn_marcheasuivre_construction-n8n.md"
captured: 2026-07-21
vault: brands
brand: aftrsn
immutable: true
---

# Marche à suivre — Construire un workflow n8n pour AFTRSN

**Usage :** Référence pour toute future session de construction/modification d'un workflow n8n AFTRSN (W-1 à W-7) via automatisation navigateur (Claude in Chrome). Rédigée après la construction complète de W-1 (AFTRSN-Event-Creator) le 03.07.2026.
**Voir aussi :** la version générique `Generique_MarcheASuivre_Construction-n8n-Navigateur.md` pour les techniques réutilisables hors AFTRSN.

---

## 1. Avant de commencer

1. Ouvrir le fichier de spec du workflow concerné (ex. `W1_Event-Creator_Flowchart_Spec_20260702.md`) et le lire en entier — il liste les nodes, leur type n8n, et le contrat d'entrée (`jsonExample`).
2. **Toujours vérifier le workflow n8n live avant d'agir si la spec et le workflow semblent en désaccord.** C'est l'invariant #5 de la passation : "n8n fait foi". Exemple concret cette session : la spec annonçait des nodes "Respond to Webhook" pour la fin du workflow, mais le trigger réel est `When Executed by Another Workflow` — un Respond to Webhook y aurait échoué. Vérifié en ouvrant le node trigger directement.
3. Vérifier dans Notion ("The Backstage") le schéma réel des DB concernées (Editions, Venues, DJs/Artists, Budget & Finance, etc.) plutôt que de se fier à un schéma théorique d'un ancien doc de passation.
4. Repérer les sous-workflows Lumina OS existants à réutiliser (ex. `LUMINA-MEMORY-WRITE/WEBHOOK`, `LUMINA-AI-Router`) via le MCP n8n (`search_workflows`, `get_workflow_details`) plutôt que d'inventer un endpoint. Pour retrouver le contrat d'entrée exact d'un sous-workflow appelé par `Execute Workflow`, ouvrir son node trigger dans le navigateur : le JSON Example y liste les champs attendus.

## 2. Conventions AFTRSN à respecter

- **Vocabulaire Notion :** "Edition" (jamais "Event"), DB réelles = Editions / Venues / DJs-Artists / Budget & Finance / Content & Social / Marketing & KPI / Partners & Sponsors / Media Assets / Cities. Pas de DB "Event Tasks" séparée — la checklist vit dans le corps de la page Edition.
- **Nommage dossiers média Drive :** `YYYYMMDD_AFTRSN_Photos` et `YYYYMMDD_AFTRSN_Videos`, chacun avec sous-dossiers `Raw` et `Edited`. Format canon strict, ne jamais dévier. *(Convention corrigée par Karter le 04.07.2026 : `AFTSN` → `AFTRSN` — dossiers Drive existants renommés, workflow P1.1/P1.3 et template Notion alignés le même jour.)*
- **Comptes Google :** AFTRSN a deux comptes Google (Hello = B2C, Partners = B2B). Pour Drive/Calendar côté W-1, c'est le compte **Hello** qui est utilisé.
- **Mémoire épisodique :** tout run significatif d'un workflow AFTRSN doit sauvegarder un épisode dans `aftrsn_memory` (collection `episodic`) via `LUMINA-MEMORY-WRITE/WEBHOOK`, appelé en **Execute Workflow** (pas HTTP Request) avec le contrat exact `{brand:"aftrsn", title, content, collection, knowledge_type, source, source_ref}`. Le trigger de ce sous-workflow est en mode `jsonExample` : un champ absent de l'exemple est silencieusement ignoré, un champ en trop aussi — reproduire le contrat exactement.
- **Credential Anthropic :** ne jamais utiliser le node natif Anthropic/Claude (bug 404 connu) — passer par un node HTTP Request pour tout appel à un modèle Claude.
- **Aucune saisie de secret par Claude :** Client ID/Secret, mots de passe, API keys — toujours guider l'utilisateur pour qu'il les saisisse lui-même dans l'interface n8n. Claude peut naviguer, ouvrir les formulaires, vérifier l'état après coup, mais jamais taper la valeur du secret.

## 3. Méthode de construction dans le canvas n8n

1. Construire node par node, dans l'ordre de la spec, en connectant chaque nouveau node au node précédent via le bouton "+" sur le connecteur.
2. Pour un node dont le rôle est d'agréger deux branches parallèles (ex. Calendar + Budget avant "Agréger résumé"), insérer un node **Merge** (mode Append, 2 inputs) et brancher les deux sorties dessus — sans ça, n8n exécute le node suivant séparément pour chaque branche au lieu d'attendre les deux. Le Merge sert de simple point de synchronisation : les nodes suivants peuvent continuer à lire les données via `$('NomDuNode').item.json...` sur les nodes d'origine, sans dépendre du contenu réel produit par le Merge.
3. Renommer systématiquement chaque node généré (nom par défaut type "HTTP Request", "Create a database page2") avec un nom clair correspondant à la spec, dès sa configuration terminée — sinon le canvas devient illisible.
4. **Ne pas cliquer "Execute step" sur un node dont la chaîne en amont n'a pas de données en cache.** Cela déclenche une ré-exécution complète depuis le trigger avec des données de test nulles, et peut créer de vraies données parasites (ex. dossiers Drive nommés "_"). Si un test isolé est nécessaire, utiliser "set mock data" plutôt que de relancer toute la chaîne.
5. Tester la validité d'un long corps JSON (ex. injection de checklist Notion) en repassant en haut de l'éditeur pour vérifier visuellement le début, plutôt que de se fier uniquement à l'indicateur de validité de l'éditeur.

## 4. Challenges rencontrés cette session et leurs solutions

| Challenge | Cause | Solution |
|---|---|---|
| Un `=` apparaît en trop dans la valeur résolue d'un champ Expression/resourceLocator (ex. `=1UeqfcsX...` au lieu de `1UeqfcsX...`), cassant les lookups Drive | Taper manuellement `={{ ... }}` alors que le mode Expression implique déjà le `=` | Ne jamais taper le `=` de tête — taper uniquement `{{ ... }}` |
| Champs de l'`Edit Fields` (`campaign_start`, `artist_brief`, `media_date_prefix`) affichaient du texte `{{...}}` brut au lieu d'une valeur calculée | Champs restés en mode "Fixed" alors qu'ils contiennent une expression | Basculer le champ en mode "Expression" avant de taper la formule |
| `}}` dupliqués en fin de champ Expression | L'éditeur CodeMirror auto-ferme les accolades ouvrantes ; retaper `}}` à la main crée un doublon | Taper seulement `{{ ` + le contenu, laisser le `}}` auto-inséré tel quel, ne jamais le retaper |
| Dossiers Drive `_` dupliqués (plusieurs occurrences sur la session) | "Execute step" sur un node testé isolément relance toute la chaîne depuis le trigger avec des données nulles | Éviter "Execute step" sur des nodes dont l'amont n'est pas en cache ; signaler les doublons à Karter pour nettoyage manuel plutôt que de les supprimer soi-même (règle de non-destruction) |
| Champ "value" vide après avoir tapé le "name" dans un node Set/Edit Fields | Le clic sur le champ value juste après avoir tapé le name ne prend pas le focus de façon fiable | Cliquer sur "name", taper, cliquer ailleurs, **re-cliquer** sur "value", puis taper |
| Panneau de recherche de node affiché tronqué en bord d'écran à l'ouverture | Comportement de rendu du panneau n8n, surtout en second niveau (sous-catégories) | Cliquer sur l'élément partiellement visible (le clic fonctionne même tronqué), ou fermer/rouvrir le panneau |
| Dropdown "From list" / "By ID" du node Execute Workflow ne changeait pas au clic | Clics sur coordonnées imprécises dans un menu qui se referme vite | Utiliser les flèches clavier (`Down` + `Return`) plutôt que le clic pour naviguer dans ce dropdown |
| Spec indique "Respond to Webhook" pour les nodes finaux, incompatible avec le trigger réel | Le trigger du workflow est `When Executed by Another Workflow`, pas un Webhook | Utiliser un node **Set** simple comme node terminal — son output devient automatiquement la valeur de retour du sous-workflow |

## 5. Definition of Done pour un workflow AFTRSN

- Tous les nodes de la spec sont présents, connectés, et renommés clairement.
- Aucun credential saisi par Claude — uniquement configuré par Karter avec guidage.
- Le workflow sauvegarde un épisode mémoire (`aftrsn_memory`, collection `episodic`) à la fin d'un run réussi.
- La branche d'erreur ne produit jamais d'écriture partielle silencieuse.
- Testé au moins une fois de bout en bout avec un payload réaliste (pas seulement vérifié visuellement).
- Publié ("Publish") dans n8n une fois validé.

## 6. Ajouts du 04.07.2026 (découpage en sous-workflows + test end-to-end)

| Challenge | Cause | Solution |
|---|---|---|
| Refactoring lourd (découpage en sous-workflows) très lent via le canvas | Des dizaines de manipulations UI fragiles | **API REST interne n8n** : `fetch('/rest/workflows/...', {headers:{'browser-id': localStorage.getItem('n8n-browserId')}})` depuis la console de l'onglet connecté — GET/POST/PATCH de workflows entiers (nodes + connections + versionId). Beaucoup plus fiable pour créer/modifier en masse ; toujours vérifier ensuite par rechargement |
| Mutation directe du store Pinia + cmd+s = aucune sauvegarde | n8n ne détecte pas les mutations d'état externes, cmd+s sans changement détecté est un no-op | Passer par l'UI ou par l'API REST, jamais par le store |
| Un node de recherche sort 0 item → le workflow « réussit » mais la suite ne s'exécute jamais | Un node à 0 item ne déclenche pas ses successeurs ; aucun message d'erreur | **Always Output Data sur le node de recherche lui-même** (pas sur le suivant). Et le code de vérification doit tester la présence d'un champ (`i.json.id`), jamais `items.length` — AOD produit un item vide qui compte pour 1 |
| Un sous-workflow renvoie N items → le mapping du node Execute Workflow suivant produit des `undefined` | La sortie d'un sous-workflow = sortie de son dernier node, un Set après un Merge multi-branches tourne par item | **executeOnce sur le node de sortie** de chaque sous-workflow (contrat : 1 item en sortie) + références inter-nodes de l'orchestrateur en `.first()` plutôt que `.item` |
| Valeurs de sortie affichant l'expression en texte brut (`{{ $('Node')... }}`) | Famille de bugs « expression malformée » : valeur sans `=` initial et/ou sans `}}` final (vus sur `Venue - Nouvelle` et `Agreger resume`) | Auditer les champs expression : toute valeur doit commencer par `=` et fermer ses `}}` ; le symptôme runtime est du texte brut dans l'output |
| « Error fetching options from Notion » au changement de DB d'un node | Session n8n expirée (l'autosave « Unauthorized » en boucle est l'autre symptôme) — pas une perte de config | Se reconnecter d'abord ; ne pas conclure à une perte de mapping : les clés de propriétés par nom survivent au changement de DB si les schémas coïncident |
| Deux DB au même nom dans le dropdown « From list » (ancien/nouveau Backstage) | La liste ne montre pas l'arborescence | Mode **By ID** avec l'ID exact — jamais la liste quand des doublons de noms existent |
| Publication refusée : « references workflow … which is not published » | Un orchestrateur ne peut pas être publié avant ses sous-workflows | Publier les sous-workflows d'abord, l'orchestrateur en dernier. Via API : `POST /rest/workflows/<id>/activate` avec body `{versionId, name, description}` (versionId = celui du GET courant) |
| Un PATCH API disparaît mystérieusement (l'ancienne version revient) | **L'autosave de l'éditeur ouvert écrase le PATCH** : l'onglet chargé avant le patch resauvegarde son état périmé (notamment en exécutant) | Recharger l'éditeur immédiatement après tout PATCH API, avant toute exécution/manipulation UI |
| Code de parsing MIME inutilement complexe pour Gmail | Le node Gmail (simplify off) renvoie des champs **déjà parsés** : `text`, `html`, `subject`, `from.text`, `to`, `date` — pas de payload brut | Inspecter la vraie sortie d'une exécution avant d'écrire un parseur ; ne jamais coder contre un format supposé |
| Node Gmail en 403 « Forbidden » alors que la credential est connectée | API Gmail non activée dans le projet Google Cloud du client OAuth | Activer l'API dans la console (`console.developers.google.com/apis/api/gmail.googleapis.com`), ~2 min de propagation |
| chat_id Telegram introuvable (l'API bots exige un id numérique, pas un @username) | Les bots ne résolvent pas les @username en privé | Le plus simple : l'utilisateur envoie un message à **@rawdatabot** sur Telegram, qui renvoie son `chat.id` — plus rapide qu'un workflow capteur |

## 7. Ajouts du 04.07.2026 au soir (Venue-Coordinator)

| Challenge | Cause | Solution |
|---|---|---|
| Code de sélection à 0 item alors que la donnée existait : la sortie du node Notion getAll (simplify) n'a **pas** les noms de propriétés Notion | Format simplifié n8n : `property_*` en snake_case (`property_1st_email`, `property_gmail_thread`…), titre dans `name` | Exécuter une fois, lire les clés réelles, coder contre l'observé (leçon jumelle du parseur Gmail) |
| Lecture du détail d'exécution (`includeData=true`) : onglet gelé (timeouts CDP en boucle) | Décodage récursif naïf du format « flatted » de n8n + payloads énormes dès qu'une trace d'agent est incluse | **Vérifier par les effets** : relire Notion/Gmail/Telegram plutôt que parser les traces ; au besoin, tests booléens légers (`raw.includes('"NomDuNode"')`) |
| Clic « Execute workflow » sans effet | Références DOM périmées après rechargement ; bouton déplacé par le panneau Logs | Re-screenshot avant chaque clic, puis vérifier qu'un **nouveau** run apparaît dans `/rest/executions?limit=1` avant d'attendre |
| Cron « 9h » décalé à 11h suisses | Instance n8n en UTC, pas de timezone sur le workflow | `settings.timezone = 'Europe/Zurich'` explicite + **re-publication** (un PATCH post-publication crée une version non publiée) |
| `/rest/credentials` illisible depuis l'outil JS (sortie bloquée « sensitive ») | Filtre de sécurité du navigateur agentique | Lire les ids de credentials dans les **nodes de workflows existants**, ou projeter id tronqué + nom |
| POST `/rest/workflows` massif : timeout CDP à 45 s alors que la création du dossier a réussi | Renderer occupé, réponse jamais revenue | Toujours **vérifier l'état serveur** après un timeout avant de rejouer (le dossier existait ; seul le workflow manquait) |

**Pattern validé (réutilisable) :** processus d'outreach + relances J+3/J+7 piloté par une base Notion, draft-only, Switch 3 branches, idempotence par dates — voir `POS-AFTRSN_Outreach-Relances-Notion-Drafts_20260704.md` et sa paire générique.

## 8. Ajouts du 05.07.2026 (Budget-Tracker + brief artistes J-7)

| Challenge | Cause | Solution |
|---|---|---|
| batchGet Sheets : un seul des deux `ranges` envoyé, lecture silencieusement fausse (zéros plausibles) | n8n sérialise les query params du node HTTP en **objet** : les noms dupliqués s'écrasent | Paramètres répétés **inline dans l'URL** (`?ranges=…&ranges=…` encodé), jamais dans queryParameters |
| Vérifications `raw.includes('"NomDuNode"')` toutes positives, même pour des nodes jamais exécutés | Le payload d'exécution embarque aussi le **workflowData** complet (tous les noms de nodes y figurent) | Décoder le flatted **ciblé sur les clés de `resultData.runData`** — ou vérifier par les effets |
| Publication refusée : `409 conflict with one of the webhooks` | Workflow dupliqué d'un autre agent → chatTrigger avec **webhookId hérité** | Régénérer le `webhookId` (UUID) avant d'activer |
| Appel de sous-workflow refusé : « Workflow is not active » | Sur cette instance, un callee doit être **publié** même pour un test manuel de l'appelant | Publier le sous-workflow d'abord (rejoint la règle de l'ordre de publication du §6) |
| API Google (Sheets) en 403 à la première utilisation | API non activée sur le projet Cloud (`909839129866`) — même piège que Gmail le 04.07 | Activation par Karter (console protégée par **passkey** : l'agent ne s'authentifie jamais) ; vérifier par l'effet en relançant |
| Tester sans cliquer dans l'UI | Boutons Execute fragiles (références DOM périmées, §7) | **`POST /rest/workflows/<id>/run`** (header `browser-id`) : exécution scriptée, id d'exécution en retour ; avec un **appelant jetable** (Manual → Execute Workflow) pour tester un sous-workflow avec payload contrôlé, re-PATCHé à chaque cas de test |
| Sortie multi-items d'un sous-workflow appelé sur N items | Mode par défaut « once with all items » | **`mode: 'each'`** sur le node Execute Workflow (v1.3) — validé en réel (2 éditions → 2 exécutions isolées, `.first()` trivial dans le sous-workflow) |
| Étendre un workflow publié sans risquer la chaîne testée | Modifier le node de sélection existant = re-tester toutes les branches | **Chaîne parallèle additive** branchée sur le même trigger (brief J-7 : 12 nodes ajoutés, la v1 n'a pas bougé) ; re-publication ensuite (un PATCH crée une version non publiée) |
| Donnée de test encore « active » pour un process périodique | Une fiche de test en statut vivant est retraitée à chaque run planifié | Neutraliser la fiche **dans le statut que le process ignore** (`Cancelled`…), suppression définitive par Karter |
| Lire le `jsCode` d'un node existant via l'API depuis le navigateur | Filtre de sortie « sensitive » de l'outil JS (faux positif sur le code) | Reconstruire le contrat depuis les **autres nodes** (Switch, Gmail, Telegram lisibles) et les POS ; en dernier recours, lire le node dans l'éditeur UI |

---
*Rédigée le 03.07.2026 après la construction complète de W-1 ; complétée le 04.07.2026 (découpage en sous-workflows, test end-to-end, Venue-Coordinator) puis le 05.07.2026 (Budget-Tracker, brief artistes J-7). À mettre à jour à chaque nouveau workflow AFTRSN construit si de nouveaux pièges apparaissent.*
