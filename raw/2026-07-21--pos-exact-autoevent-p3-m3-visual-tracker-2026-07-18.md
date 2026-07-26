---
type: raw
title: "POS-EXACT_AutoEvent-P3_M3-visual-tracker_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m3-visual-tracker_2026-07-18.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Automatisation Événementielle P3 · M3 — Visual Tracker (checklist visuels → Media Assets)

**Date :** 18.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Milestone :** M3 · **Statut :** ✅ TESTÉ, VALIDÉ & ACTIF

---

## 1. Objectif (tel que tranché avec Karter)

Karter est sur **Canva Pro/Teams**, pas Enterprise → les API **Autofill + Brand Templates** de Canva ne sont pas exploitables en production (essai « à vie » minuscule réservé au dev, quota `trial_information.uses_remaining`, puis Enterprise obligatoire). **Décision : M3 n'est PAS un générateur de visuels.** Karter fait ses flyers lui-même dans Canva et les range dans **Dropbox**.

M3 devient un **tracker** : une **checklist par wave** dans la base **Media Assets** (« flyer de la wave 1 fait ? wave 2 ? »). Un runner crée automatiquement une ligne `Raw` par wave ; Karter fait le visuel, le range dans Dropbox, et passe la ligne à `Approved`. L'automate ne clôture jamais (invariant : jamais Done / Approved par l'automate).

**Périmètre v1 :** Flyer uniquement (Story / Web ajoutés plus tard). Texte seul (pas d'image injectée). Le **Campaign angle** de M2 reste le fil rouge éditorial.

## 2. Ce qui a été construit

### 2.1 Schéma Notion (additif, zéro casse)
- **Editions** (ds `61418ed4-fb84-48fc-b372-247fbabe0c33`) : ajout **`Dropbox root`** (URL). Karter y colle **une seule fois** le lien du dossier racine de l'édition (créé même vide à la création de l'édition). Source unique.
- **Media Assets** (ds `3a4dab3b-4b77-43dd-b10a-611420b942e8`) : ajout **`Wave`** (text, ouvert : Announce / Line-up reveal / Line-up update N…) et **`Format`** (select : Flyer / Story / Web). Champs déjà présents réutilisés : `Dropbox link` (url, par ligne, non utilisé comme standard), `Edition` (relation), `Status` (Raw / Edited / Approved / Hero ref), `Type` (Photo / Video / LUT / Graphic).
- **Rollup `Dropbox (edition)`** sur Media Assets : `ROLLUP('Edition', 'Dropbox root', 'show_original')` → chaque ligne affiche automatiquement le lien Dropbox de l'édition (mis une fois sur l'édition, repris partout). Testé : la ligne créée par le runner affiche bien le lien sans qu'on l'ait collé sur la ligne.
- **Vue `🎯 Visual checklist`** (view `3a124f2f-ef0c-81ca-9a6b-000c0eee2b69`) : table groupée par `Edition`, filtrée `Format = Flyer`, triée par `Wave`, colonnes Wave / Format / Status / Dropbox (edition) / Edition / Asset. C'est la checklist « fait / pas-fait ».

### 2.2 Workflow n8n
- **`AFTRSN-VISUAL-TRACKER`** = `sJQ9Usa18kceEjlt` — **ACTIF**, Error Workflow = Sentinel `xzaH0uWy0idKVphF`, 9 nœuds. Credential Notion `AFTRSN-n8n-Backstage` (`sCsj206WVkNxO0Jl`). Patron = Copy Runner M2 (`XQYvzTzFK2NLjDZz`).
- **Nœuds :** 2 Notion Triggers (Added + Updated, Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5`, poll 10 min, `simple:true`) → **Read the task** (get databasePage) → **Should track?** (code : filtre `title contient "Flyer/visual"` && `Status = To Do` && `Executor contient Agent` && event relation non vide ; extrait `editionId` = 32 derniers hex de la relation Event, `wave` = `title.split(" · ")[0]`) → **Read the edition** (get, pour le nom) → **Build asset + check** (code : `title = "{edition} · {wave} · Flyer"` + construit le filtre de recherche) → **Check existing flyer** (HTTP POST `https://api.notion.com/v1/databases/648cdeb7bd6e41139a26dcf66a3271f6/query`, auth predefinedCredentialType notionApi, header `Notion-Version: 2022-06-28`, filtre `and[Wave equals, Format=Flyer, Edition relation contains editionId]`) → **Create if new?** (code : si `results.length > 0` → `return []` = skip idempotent ; sinon passe) → **Create Media Assets row** (create databasePage : Status `Raw`, Type `Graphic`, Format `Flyer`, Wave, relation Edition = `[editionId]`, titre).

### 2.3 Idempotence
Garantie par le **Check existing flyer** : avant toute création, on interroge Media Assets sur (Edition + Wave + Format=Flyer). Si une ligne existe déjà → skip. Robuste face au trigger « Updated » qui se relance à chaque édition de tâche. **Aucune mutation de la tâche** (contrairement à M2 qui bascule la tâche To Validate) : ici la tâche reste le fil conducteur, la ligne Media Assets est le tracker de l'asset.

## 3. Tests réalisés (avant activation)
- **Harnais Manual Trigger jetable** injectant l'ID d'une tâche de test « Flyer/visual · Mini Café » (tâche `3a124f2f-ef0c-81ed-9375-e28458421856`, taguée `demo`). Test déterministe (Notion/Manual triggers non déclenchables par API → clic « Test workflow » par Karter, lecture de l'exécution par API).
- **Exéc. #15245 (création)** : tâche « Line-up reveal · Flyer/visual · Mini Café » reconnue → aucune ligne existante → **création** de « Mini Café — 26.06.2026 · Line-up reveal · Flyer » (`3a124f2f-ef0c-81cc-b3fd-ff6a1cbafa48`), Status `Raw`, rollup Dropbox déjà rempli.
- **Exéc. #15256 (idempotence)** : même tâche → `Check existing flyer` retrouve la ligne → `Create if new?` renvoie 0 item → **aucun doublon**.
- **Contrôle base** : requête Media Assets Format=Flyer → exactement 2 lignes (Announce exemple + Line-up reveal runner), 0 doublon.
- **Activation** : harnais retiré (removeNode ×2), `activateWorkflow`, revalidation 0 erreur, 2 trigger nodes.

## 4. Difficultés rencontrées
1. **Canva gated Enterprise** : la spec P3 (D4) supposait « Pro/Teams suffit » pour Autofill + Brand Templates. Vérification doc Canva : ces API exigent une **organisation Canva Enterprise** ; Pro/Teams n'a qu'un essai de dev minuscule. Le M3 « générateur » d'origine était donc irréalisable sur le plan de Karter.
2. **Deux sources possibles pour le lien Dropbox** (par ligne vs par édition) → risque de recopie et d'incohérence.
3. **Idempotence sans mutation de tâche** : M2 s'appuie sur la bascule de statut de la tâche (To Do → To Validate) pour ne pas retraiter ; ici on ne veut PAS toucher la tâche (le visuel est un travail humain), donc il fallait un autre garde-fou.
4. **Collision avec la ligne d'exemple** : la ligne « Announce » pré-créée manuellement aurait fait « skip » le test de création (l'idempotence l'aurait détectée) → le test n'aurait pas prouvé la création.

## 5. Solutions implémentées
1. **Repivot en tracker** (pas de générateur) : Karter crée les visuels dans Canva/Dropbox, M3 ne fait que suivre l'état. Zéro dépendance Canva, zéro coût, invariants respectés.
2. **Standard « source unique »** : lien collé **une fois** sur l'édition (`Dropbox root`), repris partout par **rollup** (`Dropbox (edition)`). Le champ `Dropbox link` par ligne reste dispo mais n'est plus le standard.
3. **Garde-fou d'idempotence par requête** : `Check existing flyer` interroge Media Assets (Edition+Wave+Format) avant de créer ; skip si déjà là. Aucune mutation de la tâche.
4. **Test sur la wave suivante** : la tâche de test a été basculée sur « Line-up reveal » pour créer une 2ᵉ ligne distincte (prouve la création) tout en gardant l'exemple « Announce » (prouve, au 2ᵉ run, l'idempotence).

## 6. Lessons learned
- **Vérifier les prérequis de plan d'une API tierce AVANT de construire** : une hypothèse de la spec (Canva Pro/Teams) était fausse ; 15 min de vérif doc ont évité un build inutile.
- **« Mis une fois, repris partout »** : pour tout attribut d'édition (lien Dropbox, plus tard le budget), le poser sur l'Édition et le faire remonter par rollup — jamais recopier par ligne.
- **Distinguer checklist (fil conducteur) et fiche Édition** : deux objets séparés. La checklist = liste des choses à faire + fait/pas-fait ; l'Édition = la fiche. Reliés (relation + rollup), jamais mélangés.
- **Idempotence = choisir son point d'ancrage** : mutation d'état (M2) OU requête d'existence (M3). Le 2ᵉ évite de toucher la tâche quand l'action est humaine.
- **Pièges n8n×Notion confirmés** : lecture Notion = clés `property_snake_case` + `name` = titre ; relation écriture = TABLEAU d'IDs ; Manual/Notion triggers non déclenchables par API (harnais + clic + lecture exéc.) ; corps riche / query par HTTP API Notion (`Notion-Version` header) quand le node ne suffit pas.

## 7. IDs clés M3
- n8n : `AFTRSN-VISUAL-TRACKER` = `sJQ9Usa18kceEjlt` (actif) · Error WF Sentinel `xzaH0uWy0idKVphF` · cred Notion `sCsj206WVkNxO0Jl`.
- Notion : Media Assets DB `648cdeb7-bd6e-4113-9a26-dcf66a3271f6` / ds `3a4dab3b-4b77-43dd-b10a-611420b942e8` · vue Visual checklist `3a124f2f-ef0c-81ca-9a6b-000c0eee2b69` · Editions ds `61418ed4-fb84-48fc-b372-247fbabe0c33` (champ `Dropbox root`) · Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5`.
- Dropbox : credential n8n `Dropbox account` (`bEiwgsoILK33nI50`, dropboxOAuth2Api) — **prêt, non branché** ; réservé à l'agent aval (navigation des sous-dossiers depuis la racine).

## 8. Résidus / suite
- **Nettoyage test** : tâche de test taguée `demo` (`3a124f2f…1856`) + 2 lignes Media Assets Raw sur Mini Café (Announce exemple + Line-up reveal) — à supprimer avec le lot demo Mini Café, ou à garder comme démo pilote (décision Karter).
- **Extension** : ajouter Story + Web (runner générique par format), voire un runner unique multi-format.
- **Aval** : brancher le credential Dropbox pour qu'un agent navigue le dossier racine (flyer/stories) au moment de publier (M4/M6).
- **Finances (futur)** : même dossier racine contient le budget ; la checklist pourra porter la partie budget (valeurs saisies, éventuel Excel uploadé qui met tout à jour).
