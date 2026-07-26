---
type: raw
title: "POS-EXACT_Backstage-P2_P2-1-credential-notion-n8n_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-p2_p2-1-credential-notion-n8n_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage P2 · P2-1 — Credential Notion dédié dans n8n + test de lecture

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espaces : Notion « The Backstage » (AFTER SUN PEOPLE - HQ) + n8n LUMINA OS
Rôle : marche à suivre exacte du milestone P2-1 (spec `SPEC-BUILD_Backstage-P2-integration-n8n_2026-07-17.md`). **Validé par Karter le 17.07.2026.**

---

## 0. Décision préalable (prise en cours de route)

En créant l'intégration, découverte d'une connexion existante **`AFTRN-n8n`** (celle du pipeline LUMINA : PUBLISH-NOTION → DB « AI Automation — Knowledge », credential n8n « Notion account » créé le 20.06). **Décision Karter : ne PAS la recycler** → connexion **dédiée** au Backstage, cloisonnement strict. Corollaire noté en mémoire : à la fin du build, tout ce qui concerne LUMINA déménagera dans son propre espace Notion, et le token `AFTRN-n8n` sera renommé/repointé.

## 1. Création de la connexion Notion (Karter — les secrets ne transitent jamais par l'assistant)

1. `notion.so/profile/integrations` (UI 2026 : « integrations » = **Connections**) → **New connection**.
2. Connection name : `AFTRSN-n8n-Backstage` · Authentication method : **Access token** (token statique mono-workspace, pas OAuth) · Installable in : **AFTRSN** → **Create connection**.
3. Page Configuration de la connexion : **Capabilities** = Read content + Update content + Insert content (+ Read/Insert comments cochés — sans impact) ; User capabilities = Read user info incl. emails (défaut, sans impact pour l'usage). Copier l'**Access token**.

## 2. Donner l'accès au Backstage (Karter)

Onglet **Content access** de la connexion → Edit access → chercher « back » → sélectionner **The Backstage** avec le chemin **AFTER SUN PEOPLE - HQ** (⚠️ un doublon « The Backstage » LUMINA existe dans le picker). Résultat vérifié : Content access liste « Teamspaces (1 page) → AFTER SUN PEOPLE - HQ → The Backstage ». L'accès s'hérite à toute la structure (base Tasks incluse).

## 3. Credential n8n (Karter)

n8n → Credentials → Create credential → **Notion API** → nom **`AFTRSN-n8n-Backstage`** → coller le token → Save (« Connection tested successfully »). NB : ce test vert ≠ preuve d'accès aux pages (il valide seulement l'authentification du token).

## 4. Test de lecture (navigateur piloté, workflow scratch)

1. n8n → nouveau workflow (`/workflow/new` → « My workflow 3 », id `Xbe2TS3r9AAap0Ch`).
2. Add first step → **Manual Trigger**.
3. \+ → node **Notion** → action **Get many database pages** (Resource=Database Page · Operation=Get Many).
4. **Credential : re-sélectionner `AFTRSN-n8n-Backstage`** — piège confirmé : n8n auto-assigne le premier credential compatible (« Notion account », l'ancien) dès qu'il y en a >1.
5. Database : mode **By ID** → `cd210f519f9a4d67a50ef6f50e851dc5` (l'ID de la DB Tasks, PAS l'ID de data source). Limit 50, Simplify ON.
6. **Execute step** → 1er essai : `The resource you are requesting could not be found` (voir Difficultés). 2ᵉ essai ~10 min plus tard : **succès — 5 items** (4 tâches EXAMPLE + la tâche pilote `TEST N6 - Agent task (pilot)` `3a024f2f-ef0c-8130-9816-fc227ef9c3c3`, avec Category, Notes, rollups Budget/Budget Used et Event date résolus).

## 5. État en sortie de milestone

- Connexion Notion dédiée : `AFTRSN-n8n-Backstage` (Access token, workspace AFTRSN, accès = The Backstage AFTRSN HQ uniquement).
- Credential n8n : `AFTRSN-n8n-Backstage` (Notion API) — l'ancien « Notion account » intact pour le pipeline LUMINA.
- Workflow scratch `Xbe2TS3r9AAap0Ch` conservé : il devient la base de P2-2 (`AFTRSN-BACKSTAGE-TASK-RUNNER`).
- Onglet Webhooks de la connexion : **laissé vide volontairement** (pull only au MVP ; les webhooks Notion serviront aux déclencheurs automatiques type P0 cartographie, avec un vrai endpoint `/webhook/...` n8n et vérification, en phase ultérieure).

---

## Difficultés rencontrées

1. **« No results » dans la liste des databases du node**, puis erreur `resource not found` à l'exécution alors que le Content access était correctement configuré (vérifié : « Already added » sur The Backstage AFTRSN HQ) → soupçons successifs sur le doublon LUMINA, puis sur un mauvais token dans le credential.
2. **Auto-assignation du mauvais credential** : le node Notion s'est câblé tout seul sur « Notion account » (l'ancien) — piège déjà fiché dans la Bible (Gmail ×2), confirmé pour Notion.
3. **UI Notion 2026** : « New integration » est devenu « New connection » (avec choix Access token/OAuth) — les tutoriels historiques ne correspondent plus mot à mot.
4. Fausse piste tentée par Karter : configurer l'onglet **Webhooks** de la connexion — sans rapport avec une erreur de lecture (pull), et une URL racine sans chemin `/webhook/...` n'est de toute façon pas un endpoint valide.

## Solutions implémentées

1. **Attendre la propagation** : l'accès Notion fraîchement accordé met plusieurs minutes à être reflété par l'API. Re-test à l'identique ~10 min plus tard → succès immédiat, sans changer ni token ni credential. Diagnostic final : délai de propagation, rien d'autre.
2. Contournement de la liste « From list » par le mode **By ID** avec l'ID de la DB — plus robuste que la recherche (indexation elle aussi en retard), et test plus probant.
3. Vérification méthodique en 3 couches, dans l'ordre : capacités (Configuration) → périmètre (Content access) → identité du token (credential) — en s'arrêtant dès qu'une couche est prouvée bonne.
4. Webhooks : reportés explicitement à la phase des déclencheurs automatiques (documenté dans la spec P2 §3 et ici).

## Lessons learned

1. **Accès Notion fraîchement accordé ≠ accès immédiatement visible par l'API** : toujours attendre quelques minutes et re-tester à l'identique AVANT de changer quoi que ce soit — sinon on « répare » ce qui n'est pas cassé et on perd le diagnostic.
2. **« Connection tested successfully » d'un credential ne teste que l'authentification**, jamais le périmètre d'accès. Le seul vrai test = une lecture réelle sur la ressource cible (d'où le workflow scratch du protocole).
3. **Capabilities / Content access / token = 3 couches indépendantes** — savoir laquelle est en cause en la testant isolément évite les allers-retours. Et avec >1 credential du même type, TOUJOURS vérifier lequel le node a auto-choisi.
