---
type: raw
title: "POS-EXACT_AutoEvent-P3_M2-copy-runner_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m2-copy-runner_2026-07-18.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · M2 · AFTRSN-COPY-RUNNER (brouillons de copy → Content & Social)

**Date :** 18.07.2026 · **Marque :** AFTRSN · **Statut :** ✅ TERMINÉ, TESTÉ, VALIDÉ & ACTIF · **Auteur :** session build 18.07.

## Objectif
À partir de la tâche « Copy & captions » posée par l'Edition Watcher (M1), faire rédiger par l'agent LLM **Channel Content** les brouillons de copy multi-plateformes d'une édition, et les déposer dans **Content & Social** en `To Validate`. Interne, faible risque, aucun envoi.

## Ce qui a été construit
- **Workflow n8n `AFTRSN-COPY-RUNNER`** (`XQYvzTzFK2NLjDZz`, 14 nodes, **actif**, poll 10 min, Error WF = Sentinel `xzaH0uWy0idKVphF`).
- **Runner séparé** (Option A, pattern TASK-RUNNER Phase 2) — décision tranchée avec Karter contre l'extension du watcher, pour garder le watcher déterministe (« watcher, pas un agent »).
- Chaîne : `Notion Trigger Task Added`+`Task Updated` (base Tasks `cd210f519f9a4d67a50ef6f50e851dc5`) → `Read the task` → `Should draft?` (Code : ne garde que titre contenant « Copy & captions » + Status `To Do` + Executor `🤖 Agent` + relation Event présente ; sinon `[]`) → `Read the edition` (via relation Event) → `Build the query` (Code : reconstruit le brief, consigne EN + standard de contenu) → `Draft the copy` (Execute Sub-workflow → Channel Content `MWUwLi3vu2Ws65lL`, **entrée `query`**) → `Draft OK?` (Code : parse JSON strict + garde-fou) → `If draft OK` → **OK** : `Write draft to Content & Social` (create, 8 propriétés) → `Build body blocks` (Code) → `Append body blocks` (HTTP Notion API, dividers) → `Mark task To Validate` · **KO** : `Mark task Blocked` (Status Blocked + raison dans Notes, rien d'écrit).
- **Idempotence + porte** sans champ caché : la tâche passe **To Do → To Validate** une fois le pack posé (le runner ne prend que les `To Do` → 0 doublon au re-poll). Jamais « Done ».

## Modifs schéma Notion (Content & Social ds `e9960adf-a0c2-4947-947f-23fa65fe9cfe`, DB `709b816fe2f84dbf90ee597803a11158`)
- Statut **To Validate** ajouté (pink) — n'existait pas.
- **8 champs texte** : Campaign angle · Wix short description · Wix long description · IG caption · IG hashtags · Newsletter subject · Newsletter preview · Newsletter body.
- Corps de page (mise en page validée Karter) : sections Campaign angle / Wix · Event page / Instagram / Newsletter, avec **séparateurs + espaces**, posé par l'API Notion directe (le node n8n ne gère pas les dividers).

## Test de validation
Exécution manuelle #14937 (harness de test temporaire pointant la tâche « Announce · Copy & captions · Mini Café ») : page **« Copy pack · Mini Café · Announce »** créée en `To Validate`, 8 champs remplis, copy anglaise, ton teaser (sans line-up), 5 hashtags, cohérence cross-plateforme confirmée ; tâche passée en `To Validate`. **Validé par Karter.** Harness retiré, workflow activé.

## Difficultés
1. Contrat d'entrée de Channel Content mal noté dans la passation (« message »).
2. Le statut « To Validate » et un stockage texte long n'existaient pas dans Content & Social.
3. La cohérence entre plateformes n'était pas garantie (chaque texte indépendant).
4. « Fetch Test Event » du Notion Trigger amorçait sur une tâche au hasard (test non déterministe).
5. Impossible de taguer « demo » : le multi_select ne s'auto-crée pas via l'API page.
6. Le node Notion de n8n ne sait pas insérer de séparateur (« divider ») pour reproduire la mise en page de Karter.

## Solutions
1. Lecture directe du sous-workflow Channel Content → vrai contrat = **champ `query`** (`{{ $json.query || $json.chatInput }}`). Corrigé partout.
2. Ajout additif : statut **To Validate** + **7 champs plateforme** (puis +Campaign angle), tirés de la **doc marketing** (voix + §6 Channel & Content). Standard consigné dans `STANDARD_Contenu-par-plateforme_AFTRSN_2026-07-18.md`.
3. Ajout d'un champ **Campaign angle** : l'agent fixe d'abord une idée maîtresse (angle + émotion), puis la décline sur les 7 textes (même message, même ressenti, forme adaptée).
4. **Harness Manual Trigger** temporaire injectant l'ID de la tâche cible → test déterministe, retiré après validation.
5. Ajout de l'option `demo` au champ Tags via `ALTER … SET MULTI_SELECT(… , 'demo':brown)` avant de taguer les 20 tâches de démo Mini Café.
6. Corps de page riche posé par un node **HTTP Request → API Notion** (`PATCH /v1/blocks/{id}/children`, auth predefinedCredentialType `notionApi`, header Notion-Version), en **`onError: continueRegularOutput`** (non bloquant ; les propriétés restent la source de vérité).

## Lessons
- **Toujours vérifier le vrai contrat d'entrée d'un sous-workflow** (nom du champ de l'executeWorkflowTrigger) avant de câbler un Execute Sub-workflow ; ne pas se fier à une passation.
- **Standardiser le contenu par plateforme AVANT de générer** : un modèle par canal + une idée maîtresse commune = cohérence de campagne. Un agent LLM livre du texte libre → imposer un **JSON strict** + garde-fou (Blocked si non conforme).
- **La porte de validation peut porter l'idempotence** : To Do → To Validate suffit, pas besoin de champ caché.
- **Notion API par HTTP dans n8n** = échappatoire propre quand le node natif ne couvre pas un type de bloc (divider, callout…), à mettre en non bloquant.
- Pour tester un Notion Trigger de façon déterministe : harness Manual Trigger jetable.

## IDs clés
n8n : COPY-RUNNER `XQYvzTzFK2NLjDZz` · Channel Content `MWUwLi3vu2Ws65lL` · Edition Watcher `git92S6gDU1hGZ2b` · Sentinel `xzaH0uWy0idKVphF` · cred Notion `sCsj206WVkNxO0Jl`.
Notion : Tasks ds `e50f931d-3f8d-483f-895a-c09408455128` · Content & Social ds `e9960adf-a0c2-4947-947f-23fa65fe9cfe` (DB `709b816fe2f84dbf90ee597803a11158`) · Editions ds `61418ed4-fb84-48fc-b372-247fbabe0c33` · Mini Café `39e24f2fef0c812ab3bbe8636418a482`.
