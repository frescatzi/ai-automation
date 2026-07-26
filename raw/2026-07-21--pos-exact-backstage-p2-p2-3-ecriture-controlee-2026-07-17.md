---
type: raw
title: "POS-EXACT_Backstage-P2_P2-3-ecriture-controlee_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-p2_p2-3-ecriture-controlee_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage P2 · P2-3 — Écriture contrôlée (claim + report, jamais Done)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espaces : n8n LUMINA OS + Notion Backstage (AFTER SUN PEOPLE - HQ)
Rôle : marche à suivre exacte du milestone P2-3 (spec `SPEC-BUILD_Backstage-P2-integration-n8n_2026-07-17.md`). **Validé par Karter le 17.07.2026.**

---

## 1. Architecture ajoutée (workflow `AFTRSN-BACKSTAGE-TASK-RUNNER` `Xbe2TS3r9AAap0Ch`)

Chaîne complète après P2-3 :
`Manual Trigger` → `Read agent tasks` → `Select and map the pilot task` → **`Claim the task`** → **`Report back`**

**Invariant câblé physiquement : aucun node ne connaît la valeur « Done »** — le workflow ne peut écrire que `In Progress` et `To Validate`. Anti double-run : le garde-fou amont n'accepte que Status=To Do, donc une tâche déjà In Progress n'est jamais re-saisie.

## 2. Node `Claim the task` (Notion → Update a database page)

- Credential : `AFTRSN-n8n-Backstage` (⚠️ re-sélectionner — l'auto-assignation remet « Notion account »).
- Resource=Database Page · Operation=Update.
- **Database Page : mode By URL + champ en Expression** : `https://www.notion.so/{{ $json.page_id }}`.
  ⚠️ Piège découvert : le node ne sait PAS parser les URLs `app.notion.com/p/...` (« Could not extract page ID from URL ») — il faut une URL `www.notion.so/<id>`. D'où la construction manuelle. (Le sélecteur By ID refusait le clic — contournement équivalent.)
- **Properties** : la liste d'options ne charge pas quand la page est une expression (« Error fetching options from Notion » — normal, rien à réparer) → passer **Key Name or ID en Expression** : `Status|select` (format `nom|type`), puis le champ **Option Name or ID en Expression** : `In Progress`.
- Test isolé : Execute step → succès ; vérifié par requête SQL Notion : Status=In Progress. ✓

## 3. Node `Report back` (Notion → Update a database page)

- Credential : `AFTRSN-n8n-Backstage` (re-sélectionné, même piège).
- Database Page (By URL, Expression) : `https://www.notion.so/{{ $json.id || $json.page_id }}` — **le `||` est important** : l'input vient de la sortie de Claim (champ `id`) mais si Claim est traversé sans exécuter (désactivé/passthrough), c'est `page_id` du mapping qui arrive.
- Properties ×2 (Key en Expression) :
  1. `Status|select` → Option : `To Validate`.
  2. `Notes|rich_text` → Rich Text OFF → Text : `[P2-3 TEST] Placeholder report written by AFTRSN-BACKSTAGE-TASK-RUNNER. Chain verified: To Do -> In Progress -> To Validate. Real content will come from Hermes in P2-4. Awaiting founder validation.`

## 4. Test bout-en-bout

1. Remise de la pilote en To Do (connecteur Notion, `update_properties`).
2. **Execute workflow** (jamais Execute step pour un test à données fraîches) → 4 nodes verts en ~2,5 s : Read (6 items) → Select (1) → Claim → Report.
3. Vérifications Notion : SQL → Status=`To Validate`, Notes=le placeholder ✓ ; query de la vue ✅ Awaiting validation (`39f24f2f-ef0c-8143-aa54-000c238a5ada`) → la pilote y figure, compte-rendu lisible ✓.
4. État de sortie : pilote en To Validate avec Notes placeholder — Karter peut la voir dans sa file founder. Pour P2-4, la remettre en To Do.

## 5. Incidents de build (résolus)

- **Node Claim désactivé par un clic parasite** sur l'icône power de la toolbar du node — détecté par le bandeau « The node is disabled » dans le panneau INPUT du node suivant ; réactivé via le lien « Enable it ».
- Ce même passthrough du node désactivé a produit l'erreur « Could not extract page ID from URL: https://www.notion.so/ » (expression `$json.id` vide) → résolu par le `||` défensif ET la réactivation.

---

## Difficultés rencontrées

1. **Parseur d'URL du node Notion limité** : les URLs `app.notion.com/p/...` (celles que renvoie l'API !) ne sont pas reconnues — seules les `www.notion.so/<id>` passent.
2. **Options de propriétés non chargées** quand la page cible est une expression → impossible de choisir Status/Notes dans les listes déroulantes.
3. **Sélecteur By URL/By ID récalcitrant** au clic automatisé (le choix ne s'appliquait pas).
4. **Clic parasite = node désactivé silencieusement** ; symptôme d'abord incompris (expression vide en aval).

## Solutions implémentées

1. Construction de l'URL en expression : `https://www.notion.so/{{ $json.id || $json.page_id }}` — insensible au format d'entrée et au passthrough.
2. **Clés de propriétés en mode Expression au format `nom|type`** (`Status|select`, `Notes|rich_text`) — les champs de valeur apparaissent alors normalement (Option Name, Text).
3. Abandon du sélecteur têtu au profit du mode déjà actif + expression équivalente.
4. Lire le bandeau d'état du panneau INPUT avant de diagnostiquer une expression : un node amont désactivé change le contrat de données.

## Lessons learned

1. **L'invariant de sécurité le plus fort est celui que l'outil ne peut pas exprimer** : ne jamais mettre « Done » dans aucun node = impossible à écrire, même par bug. Mieux qu'une consigne, mieux qu'un IF.
2. **Écrire les expressions défensivement** (`a || b`) quand l'input peut venir de deux chemins (node exécuté vs traversé) — coût nul, évite une classe entière d'erreurs.
3. Le format `propriété|type` en expression est LA clé du node Notion Update quand la cible est dynamique — à réutiliser pour toute écriture Notion paramétrée (P2-4 compris).
