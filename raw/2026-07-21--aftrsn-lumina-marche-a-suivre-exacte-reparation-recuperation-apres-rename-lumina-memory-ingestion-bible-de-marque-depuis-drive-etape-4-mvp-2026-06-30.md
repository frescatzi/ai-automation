---
type: raw
title: "AFTRSN-LUMINA — Marche a suivre EXACTE — Reparation recuperation apres rename lumina_memory + ingestion bible de marque depuis Drive (etape 4 MVP) — 2026-06-30"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — marche a suivre exacte — reparation recuperation apres rename lumina_memory + ingestion bible de marque depuis drive (etape 4 mvp) — 2026-06-30.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# Marche à suivre EXACTE — Réparer la récupération mémoire après le rename `lumina_memory` + repointer l'ingestion + ingérer la bible de marque depuis Google Drive (étape 4 MVP)

**Date :** 2026-06-30
**Système :** n8n self-hosté (`n8n.aftersunpeople.com`) + pgvector (DB `n8n`, schéma `public`) + MCP `LUMINA-MCP-Server`
**Validateur final :** Katel (Karter)
**Contexte :** la veille, `asp_memory` a été renommée `aios_memory` puis `lumina_memory` (banque partagée d'automation/process). Au démarrage de cette session, la récupération mémoire (MCP `search_brand_memory`) ne répondait plus. Objectifs : (1) réparer la récupération, (2) repointer l'ingestion sur `lumina_memory`, (3) ingérer la bible de marque dans `aftrsn_memory` (étape 4).

> Convention : explications en français ; noms de workflows/nodes/credentials et code **en anglais** (comme dans n8n). Numérotation à partir de 1.

---

## 1. Symptôme de départ

Appel du MCP `search_brand_memory(brand=lumina)` → `NodeApiError` (« The service was not able to process your request »). La récupération `LUMINA-RETRIEVAL-MEMORY/WEBHOOK` était cassée depuis le renommage de la table.

## 2. Localiser la vraie cause (méthode Executions)

1. n8n → onglet **Executions** du workflow `LUMINA-RETRIEVAL-MEMORY/WEBHOOK`.
2. Ouvrir l'exécution rouge la plus récente → le node **Webhook** affichait « No Respond to Webhook node found in the workflow ».
3. Première hypothèse (fausse) : le node `Respond to Webhook` était déconnecté.
4. Test discriminant : forcer le webhook en mode « When Last Node Finishes / All Entries » → l'appel a renvoyé **la requête SQL elle-même** (sortie du node `Build Search`), pas les résultats.

**Leçon clé :** si le webhook renvoie la sortie d'un node intermédiaire (ici la chaîne `query`), c'est que **la chaîne est coupée en aval** : le node suivant ne s'exécute jamais.

## 3. Cause racine — la connexion `Build Search → Postgres Search` était rompue

Les éditions chirurgicales de la veille (corrections du code `Build Search`) avaient **cassé le lien** `Build Search → Postgres Search`. Résultat : `Postgres Search` ne tournait jamais ; le webhook renvoyait seulement la requête construite. L'erreur « No Respond node found » n'était qu'un symptôme secondaire.

### Correctif appliqué (architecture simplifiée, plus robuste)

1. **Reconnecter** `Build Search` (sortie) → `Postgres Search` (entrée) par drag-and-drop sur le canvas.
2. Sur le node **`Webhook`** : `Respond` = **« When Last Node Finishes »**, `Response Data` = **« All Entries »**.
3. **Supprimer** le node `Respond to Webhook` (avec lui présent — même désactivé — n8n renvoyait « Unused Respond to Webhook node found in the workflow »).
4. Chaîne finale de récupération : `Webhook → Embed Question (OpenAI) → Build Search → Postgres Search` (dernier node, renvoie directement les lignes).
5. **Publish**.

**Challenge A :** l'éditeur me montrait la connexion visuellement présente alors qu'elle était rompue à l'exécution.
**Solution A :** se fier à la **sortie réelle** (Executions), pas au visuel ; recréer la connexion à la main.

**Challenge B :** garder un node `Respond to Webhook` (même désactivé) avec le mode « When Last Node Finishes » fait planter le webhook (« Unused Respond… »).
**Solution B :** supprimer entièrement le node Respond ; le mode « last node » se suffit à lui-même.

### Vérification
- MCP `search_brand_memory(brand=lumina)` → renvoie 5 lignes réelles de `lumina_memory` (66 lignes). ✅
- MCP `search_brand_memory(brand=aftrsn)` → `[{}]` (banque vide, pas de crash). ✅

## 4. Repointer l'ingestion partagée sur `lumina_memory`

Le workflow `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR` (id `OUWDFF1HUhBr0Vy1`) référençait encore `asp_memory` (table inexistante). Repointé les **3 nodes SQL** :

1. **`Truncate (SQL query)`** : `TRUNCATE asp_memory;` → `TRUNCATE lumina_memory;`
2. **`Build Insert`** (ligne 14 du code) : `"INSERT INTO asp_memory "` → `"INSERT INTO lumina_memory "`
3. **`Verify Table Count`** : `FROM asp_memory` → `FROM lumina_memory`
4. **Publish** (sans exécuter — pour ne pas TRUNCATE + ré-embedder les 66 lignes vivantes inutilement).

**Leçon :** après un rename de table, balayer **tous** les workflows (récupération **et** ingestion) ; ne pas exécuter un pipeline « TRUNCATE puis reload » juste pour le repointer.

## 5. Étape 4 — Ingérer la bible de marque dans `aftrsn_memory`

Workflow `AFTRSN-MEMORY-INGESTION/DRIVE→AFTRSN_MEMORY` (id `krWBy2Sj9k47H24z`), au départ copie exacte de l'ingestion GitHub→pgvector.

### 5.1 Découverte sur le Drive (décisive)
Le dossier source [folder `1QUp9OtmA_O4NJBwnph1ZtIcoHjmKiB0o`] est **imbriqué**, pas plat :
- Racine → sous-dossiers `AFTRSN_LOGO`, `AFTRSN_Official Documents`, `AFTRSN_Fonts`.
- `AFTRSN_Official Documents` → `Visual Concept`, `Partnership`, `Marketing`, `Brand Business` (+ 1 PDF présentation).
- `Brand Business` → la bible : **`20260215_Corporate Brand Identity.pdf`** (file id `1nJwBnEA0a7fDlSG2-NgjCagIupzn8a2e`, 319 Ko), à côté des `.docx` équivalents (ignorés : PDF/md seulement).

Une requête Drive « fichiers du dossier » (`'folderId' in parents`) ne renvoie que les **sous-dossiers** → le « loop sur dossier » ne marche pas sur une arborescence imbriquée sans récursivité.

**Décision (Karter) :** MVP = **cibler directement le PDF de la bible par son ID** ; la traversée récursive multi-fichiers passe en **étape 4b**.

### 5.2 Transformation du workflow
1. **Supprimer** les nodes source GitHub : `HTTP Request`, `Split Out`, `Get a file`, puis aussi `Search files and folders` (Drive) et `Filter` (devenus inutiles pour un fichier unique). n8n **recrée automatiquement le pont** entre les nodes restants à chaque suppression d'un node central.
2. **Insérer** sur la connexion `Truncate → Set Document` : **`Google Drive` → `Download file`**.
   - Credential : `Google Drive account - Live`.
   - `File` = **By ID**, mode **Fixed** : `1nJwBnEA0a7fDlSG2-NgjCagIupzn8a2e`.
   - Sortie binaire dans le champ **`data`**.
3. **Insérer** ensuite **`Extract from File` → `Extract From PDF`** ; `Input Binary Field` = **`data`**. (Sortie texte dans `$json.text`.)
4. **`Set Document`** (Manual Mapping) — adapter les champs (avant = GitHub) :
   - `document` = `{{ $json.text }}`  (était `{{ $json.content.base64Decode() }}`)
   - `title` = `Corporate Brand Identity`  (était `{{ $json.name }}`)
   - `source` = `Drive`  (était `GitHub`)
   - `source_ref` = `20260215_Corporate Brand Identity.pdf`  (était `{{ $json.path }}`)
   - `collection` = `canon`  (était `ai_os`)
   - `knowledge_type` = `standard` (inchangé)
5. **Retargeter les 3 nodes SQL** sur `aftrsn_memory` : `Truncate`, `Build Insert` (ligne 14), `Verify Table Count`.
6. Vérifier que **`Embed (OpenAI)`** utilise bien la credential **`OpenAi account`** (le duplicata l'avait déjà — créé après le correctif credential).

### 5.3 Exécution & vérification
1. **Execute workflow** → tous les nodes verts (6,7 s) : Truncate → Download (1 item) → Extract (1) → Set Document (1) → **Chunk (2 items)** → **Embed (2 items, 2 s)** → Build Insert (2) → Postgres Insert (2).
2. Le PDF est découpé en **2 chunks**, embeddés, insérés dans `aftrsn_memory`.
3. **Test end-to-end MCP** `search_brand_memory(brand=aftrsn)` → renvoie « Corporate Brand Identity [1]/[2] » : *Brand Name AFTER SUN PEOPLE, AFTER SUN = Day & Night Ritual celebrating African Electronic Dance, palette, design editorial…* ✅
4. **Publish** avec note de version.

**Challenge C :** les champs **Expression** de n8n auto-ferment `{{` → en tapant `{{ $json.text }}` on obtient `{{ $json.text }} }}`.
**Solution C :** après la saisie, `End` puis **3× Backspace** pour retirer le ` }}` en trop ; vérifier le **Result**.

**Challenge D :** seulement **2 chunks** pour un PDF de 319 Ko.
**Cause :** PDF très « design » → peu de texte extractible par `Extract From PDF`. La liste précise des genres (Afro House / Afro Tech / Melodic Afro House) peut être peu/pas captée.
**Piste 4b :** ingérer aussi un `.md`/`.docx` converti où ce texte est net.

## 6. Résumé des leçons apprises

- Si le webhook renvoie la sortie d'un node **intermédiaire**, la chaîne est **coupée en aval** → vérifier/recréer la connexion (ne pas se fier au visuel).
- Récupération plus robuste : `Webhook (When Last Node Finishes / All Entries)` **sans** node `Respond to Webhook` (sinon « Unused Respond… »).
- Après un rename de table : repointer **récupération + ingestion** ; ne pas exécuter un pipeline TRUNCATE-reload juste pour le repointer.
- Avant de « boucler sur un dossier Drive », **inspecter l'arborescence** : un dossier imbriqué casse la requête plate `in parents`.
- Drive → texte : `Download file (By ID, binary 'data')` → `Extract from File (Extract From PDF)` → `$json.text`.
- Champs Expression n8n : auto-close `{{ }}` → nettoyer le ` }}` en trop (End + 3 Backspace).
- Valider **end-to-end** (appel MCP réel), pas node par node.

## 7. Reste à faire

- **Étape 4b :** traversée récursive des sous-dossiers Drive (autres PDF) + branche markdown (`Extract from text file`) ; produire des `.md` propres dans Obsidian `brands/aftrsn`.
- **Câblage étape 3 :** brancher les 3 agents spécialistes sur `brand=aftrsn` ; le Superviseur reste sur `lumina`.
- **Redéfinir les agents** (en cours avec Karter).
