---
type: raw
title: "POS-EXACT_Backstage-P2_P2-2-task-runner-lecture-garde-fou_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-p2_p2-2-task-runner-lecture-garde-fou_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage P2 · P2-2 — TASK-RUNNER : lecture du monde agent + garde-fou pilote

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espaces : n8n LUMINA OS + Notion Backstage (AFTER SUN PEOPLE - HQ)
Rôle : marche à suivre exacte du milestone P2-2 (spec `SPEC-BUILD_Backstage-P2-integration-n8n_2026-07-17.md`). **Validé par Karter le 17.07.2026.**

---

## 1. Base de départ

Workflow scratch de P2-1 (`Xbe2TS3r9AAap0Ch`, « My workflow 3 ») : Manual Trigger → node Notion « Get many database pages » (cred `AFTRSN-n8n-Backstage`, DB By ID `cd210f519f9a4d67a50ef6f50e851dc5`, Limit 50, Simplify ON). Exécution navigateur piloté (session n8n active dans l'onglet ; création de workflow via URL directe `/workflow/new` fonctionne).

## 2. Renommage (convention : verbes d'action, anglais)

1. **Workflow** : clic sur le titre → cmd+A → `AFTRSN-BACKSTAGE-TASK-RUNNER` → Tab.
2. **Node Notion** : sélectionner le node → **F2** (ouvre le dialogue « Rename Node ») → `Read agent tasks` → Return.
3. **Node Code** : F2 → `Select and map the pilot task` → Return.

## 3. Node garde-fou + mapping (Code in JavaScript, Run Once for All Items)

Ajouté sur la sortie de `Read agent tasks` (+ → Code → Code in JavaScript). Contenu :

```javascript
// HARD GUARD: only the designated pilot task may pass (P2 scope)
const PILOT_ID = "3a024f2f-ef0c-8130-9816-fc227ef9c3c3";
const AGENT = "🤖 Agent";
const READY = "To Do";
const picked = [];
for (const item of $input.all()) {
  const t = item.json;
  const isPilot = t.id === PILOT_ID;
  const isAgent = t.property_executor === AGENT;
  const isReady = t.property_status === READY;
  if (isPilot && isAgent && isReady) picked.push(t);
}
return picked.map(t => ({ json: {
  page_id: t.id,
  title: t.name,
  category: t.property_category ?? null,
  event: Array.isArray(t.property_event) && t.property_event.length ? t.property_event[0] : null,
  notes: t.property_notes ?? "",
  status: t.property_status,
  executor: t.property_executor,
  url: t.url
} }));
```

Champs d'entrée (sortie Simplify du node Notion) : `id`, `name` (titre), `url`, `property_executor`, `property_status`, `property_category`, `property_event` (array), `property_notes`.

## 4. Tests (les 3 sens du garde-fou)

1. **Pilote en To Validate** → Execute step : `No output data returned` (0 item). ✓ Le runner ignore une tâche non prête.
2. **Pilote passée en To Do** (via connecteur Notion : `update_properties {Status: "To Do"}` sur `3a024f2f…c3c3`) → **Execute workflow** (⚠️ pas Execute step : le step seul rejoue l'INPUT EN CACHE de l'ancien fetch) : Notion 5 items → Code **1 item** mappé (brief complet, event = Mini Café `39e24f2f-ef0c-812a-b3bb-e8636418a482`). ✓
3. **Leurre** : création par API d'une tâche `TEST P2 - Decoy agent task` (`3a024f2f-ef0c-8191-848c-c72c00da2f5d`, Executor=🤖 Agent, Status=To Do, non-pilote) → Execute workflow : Notion **6 items** → Code **1 item** (la pilote seule, leurre ignoré). ✓ Leurre corbeillé ensuite par Karter.

## 5. Réglages de fin de milestone

- **Error Workflow** : ••• → Settings → Error Workflow = `LUMINA-SENTINEL/ERROR-WATCH` (`xzaH0uWy0idKVphF`) → Save (« Workflow settings saved »).
- **Tags** : + Add tag → `AFTRSN` + `BRAIN` + `TEST` (existants dans la taxonomie).
- Workflow **non publié** (déclencheur manuel au MVP — décision de publication en P2-6).
- Reste (manuel Karter, sans urgence) : drag-drop du workflow vers le dossier `02-AFTRSN / AFTRSN-03-BRAIN` (menus contextuels n8n instables en automatisation — piège figé 12-07).
- État de sortie : tâche pilote laissée en **To Do** (prête pour le test P2-3).

---

## Difficultés rencontrées

1. **SyntaxError `Unexpected token '}'` au premier run du Code node** : l'éditeur CodeMirror auto-ferme les accolades pendant la frappe → 2 lignes orphelines (`}}))` + `}`) ajoutées silencieusement en fin de code, invisibles sans scroller.
2. **Execute step ≠ re-fetch** : après le passage de la pilote en To Do, « Execute step » sur le Code node renvoyait toujours 0 item — il rejoue l'input EN CACHE du node amont, pas un nouveau fetch Notion.
3. Double-clic sur un node = zoom canvas (pas d'ouverture) ; le menu ••• du workflow se referme si on re-clique ; un onglet Chrome fraîchement créé est parfois resté « chrome-internal » impossible à piloter (corbeille du leurre déléguée à Karter).

## Solutions implémentées

1. Aller en fin de code (cmd+Down) → suppression des accolades orphelines (7 Backspace) → run vert. Réflexe : **toujours vérifier la FIN du code après une frappe longue dans CodeMirror**.
2. **Execute workflow** (bouton global) pour tout test qui dépend de données fraîches ; Execute step réservé au debug d'un node sur input connu.
3. F2 = renommage fiable d'un node sélectionné (évite le double-clic) ; localisation des entrées de menu par accessibilité (`find`) plutôt qu'aux pixels.

## Lessons learned

1. **Un garde-fou se prouve dans les 3 sens** : la cible ne passe pas quand elle n'est pas prête · elle passe seule quand elle l'est · un intrus conforme aux autres critères ne passe PAS. Le test du leurre est celui qui prouve vraiment le cloisonnement.
2. **Le garde-fou en dur dans le code** (ID + executor + status) est lisible, testable et impossible à contourner par une donnée — préférable à un filtre de vue pour un périmètre de sécurité.
3. Éditeurs de code web pilotés : la frappe « naïve » marche à 95 % — les 5 % restants sont TOUJOURS en fin de fichier (auto-close). Vérifier la fin avant d'exécuter.
