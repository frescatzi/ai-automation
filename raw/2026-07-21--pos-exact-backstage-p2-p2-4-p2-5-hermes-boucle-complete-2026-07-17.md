---
type: raw
title: "POS-EXACT_Backstage-P2_P2-4-P2-5-hermes-boucle-complete_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-p2_p2-4-p2-5-hermes-boucle-complete_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage P2 · P2-4 + P2-5 — Branchement Hermès + boucle complète validée (run réel + validation founder)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espaces : n8n LUMINA OS + Notion Backstage (AFTER SUN PEOPLE - HQ)
Rôle : marche à suivre exacte des milestones P2-4 (branchement du moteur Hermès) et P2-5 (run réel bout-en-bout + validation founder), spec `SPEC-BUILD_Backstage-P2-integration-n8n_2026-07-17.md`. **Validé par Karter le 17.07.2026 (pilote passée en Done à la main).**

---

## 0. État à l'entrée

Chaîne héritée de P2-3 : `Manual Trigger` → `Read agent tasks` → `Select and map the pilot task` → `Claim the task` (In Progress) → `Report back` (placeholder). Le node IF envisagé au design initial a été **supprimé** : la logique succès/échec est portée par des expressions conditionnelles dans le seul node `Report back` (design simplifié, moins de nodes fragiles au navigateur).

Workflow : `AFTRSN-BACKSTAGE-TASK-RUNNER` (`Xbe2TS3r9AAap0Ch`). Moteur : `LUMINA-Hermes-Agent` (`Dwv4rcMqNAyQzlrF`).

## 1. Node `Call 'LUMINA-Hermes-Agent'` (Execute Sub-workflow)

Intercalé entre `Claim the task` et `Report back`.

- Workflow cible : `LUMINA-Hermes-Agent` (mode list).
- **Contrat d'entrée d'Hermès** : le trigger `On Workflow Call` n'accepte qu'un champ `message` (jsonExample `{"message": "..."}` — les autres champs sont filtrés). Donc mapping `defineBelow` avec l'unique clé `message`.
- **Brief envoyé** (expression) — référence le node de mapping en amont, pas `$json` (car l'input de Call Hermès = sortie de Claim, qui ne porte pas le titre/notes) :
  ```
  ={{ "Tache AFTRSN a traiter (depuis le Backstage). Titre: " + $('Select and map the pilot task').first().json.title + ". Details: " + ($('Select and map the pilot task').first().json.notes || "aucun") + ". Produis un court compte-rendu (3-5 lignes) de ce que tu ferais concretement pour accomplir cette tache." }}
  ```
- **Contrat de sortie d'Hermès** : `{ text, outcome, usedTools }`, `outcome ∈ success | fail | timeout`.

## 2. Node `Report back` — expressions conditionnelles (le cœur de P2-4)

L'input de `Report back` = **sortie d'Hermès** → donc `$json.outcome` et `$json.text` référencent directement le résultat d'Hermès (pas besoin de re-nommer le node Hermès dans l'expression, ce qui évite le piège du guillemet simple dans « Call 'LUMINA-Hermes-Agent' »).

- Database Page (By URL, Expression) — la cible ne vient PAS de `$json` (qui est Hermès, sans page_id) mais du mapping d'origine :
  `=https://www.notion.so/{{ $('Select and map the pilot task').first().json.page_id }}`
- Property `Status|select` (clé en Expression) → Option en Expression :
  `={{ $json.outcome === "success" ? "To Validate" : "Blocked" }}`
- Property `Notes|rich_text` (clé en Expression, Rich Text OFF) → Text en Expression :
  `={{ ($json.outcome === "success" ? "" : "[BLOCKED] ") + $json.text }}`

**Invariant câblé physiquement** : aucune branche ne peut écrire `Done`. Succès → `To Validate` (file founder) ; échec/timeout → `Blocked` + compte-rendu préfixé `[BLOCKED]`. La clôture reste 100 % humaine.

## 3. P2-5 — Run réel bout-en-bout

1. Remise de la pilote (`3a024f2f-ef0c-8130-9816-fc227ef9c3c3`) en **To Do** (connecteur Notion `update_properties`).
2. **Execute workflow** (données fraîches) → **succès en 33,6 s**, 6 nodes verts. Read (6 items) → Select (1) → Claim (In Progress) → Call Hermès (`outcome: success`, ~30 s de travail LLM réel) → Report back (804 ms).
3. Vérification API Notion (`notion-fetch`) : Status = **To Validate**, Notes = le vrai compte-rendu d'Hermès (plan en 4 points 📝🛠️🔁✅), **sans** préfixe `[BLOCKED]` (cohérent avec `outcome=success`). Executor inchangé 🤖 Agent.
4. **Validation founder** : Karter lit la pilote dans la vue ✅ Awaiting validation et la passe en **Done**. Confirmé par API : `Status: Done`. La boucle Backstage↔Agent↔Founder est bouclée.

## 4. Correctif post-run par API (nouvel accès écriture n8n-mcp)

Le pont MCP local `n8n-mcp` (accès écriture complet à l'API n8n, actif le 17.07) a permis de lire le JSON authentique du workflow — invisible aux captures navigateur. Il a révélé un **`}}` surnuméraire en fin d'expression du `message`** de Call Hermès (`... cette tache." }} }}`), vestige de l'auto-fermeture CodeMirror du build navigateur d'une session antérieure. Effet réel : le ` }}` littéral était **concaténé en fin de message** envoyé à Hermès — toléré (Hermès l'a ignoré), mais sale.

- Correctif : `n8n_update_partial_workflow` → `patchNodeField` sur `parameters.workflowInputs.value.message`, find ` }} }}` → replace ` }}`.
- Re-vérification : `n8n_get_workflow` (message se termine sur `cette tache." }}`) + `n8n_validate_workflow` → `valid: true`, 0 erreur, 0 warning, 5 expressions validées.
- **Pas de re-run** : le fix ne retire qu'un littéral de fin de message (l'expression ne peut que s'améliorer), et on préserve l'état **Done** validé par le founder. Un run antérieur fonctionnellement équivalent (message + « }} » parasite) avait déjà réussi ; la version nettoyée est strictement meilleure.

## 5. Validation croisée finale (API)

- `n8n_validate_workflow` : 6 nodes, 6 activés, 1 trigger, 5 connexions valides / 0 invalide, 5 expressions validées, 0 erreur, 0 warning.
- Error Workflow global déjà posé en P2-2 (`LUMINA-SENTINEL/ERROR-WATCH`) → la suggestion générique « ajoutez onError par node » est couverte au niveau workflow.
- Workflow toujours **non publié** (déclencheur manuel) — la décision d'automatisation (trigger planifié/webhook) est un chantier ultérieur, hors P2 pilote.

---

## Difficultés rencontrées

1. **Référencer le bon node dans les expressions** : après insertion de Call Hermès, l'input de Report back change de contrat (Hermès, pas la tâche mappée) — `$json` ne porte plus `page_id`.
2. **Guillemet simple dans le nom du node** « Call 'LUMINA-Hermes-Agent' » : rend `$("Call '...'")` pénible et fragile à taper dans CodeMirror.
3. **Auto-fermeture CodeMirror** sur les expressions complexes (guillemets, parenthèses, accolades) — source du `}}` surnuméraire hérité.
4. Le run réel dépend d'un vrai appel LLM (~30 s) : ne pas confondre « lent » et « bloqué ».

## Solutions implémentées

1. Cible de Report back référencée explicitement via `$('Select and map the pilot task').first().json.page_id` ; outcome/text via `$json` (= sortie Hermès directe).
2. **Utiliser `$json`** plutôt que `$("Call '...'")` : l'input direct du node porte déjà la sortie d'Hermès → zéro guillemet simple à échapper.
3. Frappe sans accolades fermantes (auto-close finit le travail) + vérification typeover des guillemets par capture ; **correctif définitif par API** une fois le pont n8n-mcp disponible (find/replace chirurgical au lieu de re-typer dans le navigateur).
4. Attente active (10 s + 10 s) puis lecture de l'état ▶️ du titre d'onglet ; le node Hermès rouge = en cours, pas en erreur.

## Lessons learned

1. **L'input direct d'un node EST son `$json`** : quand un moteur (sous-workflow) précède le node d'écriture, référencer sa sortie via `$json` est plus court, plus lisible et évite d'échapper un nom de node à caractères spéciaux. Ne référencer un node par son nom que pour les données qu'il ne porte PAS.
2. **`outcome`→statut est le mapping qui incarne « l'agent propose, l'humain dispose »** : succès = file de validation, échec = bloqué + raison, jamais terminé. Une seule expression ternaire suffit — pas besoin d'un node IF fragile.
3. **Le JSON par API est la source de vérité, pas la capture d'écran** : un défaut (le `}}` surnuméraire) invisible à l'œil et toléré à l'exécution n'apparaît qu'en lisant la config réelle. Dès qu'un accès API existe, relire et valider par API après tout build navigateur.
4. **Ne pas re-run pour un fix cosmétique validé** : préserver un état humainement validé (Done) vaut mieux qu'un re-test qui le détruit, quand la validation statique prouve déjà l'innocuité.
