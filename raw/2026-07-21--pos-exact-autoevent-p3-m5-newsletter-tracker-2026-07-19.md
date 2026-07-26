---
type: raw
title: "POS-EXACT_AutoEvent-P3_M5-newsletter-tracker_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m5-newsletter-tracker_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent P3 · M5 — Newsletter Tracker (`AFTRSN-NEWSLETTER-TRACKER`)

**Date :** 19.07.2026 · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Type :** POS-EXACT (procédure exacte, reproductible) · **Statut :** ✅ CONSTRUIT, TESTÉ LIVE, VALIDÉ (Karter, 19.07), ACTIF.

---

## 0. Résumé
`AFTRSN-NEWSLETTER-TRACKER` (`HnX3llf1rW6qmlPb`) est un **runner déterministe** (scan 1 min) qui transforme la case de checklist « Newsletter » d'une édition en un livrable **prêt à coller** : il va chercher la copie **approuvée** de la vague dans Content & Social, assemble sujet + preview + corps dans le champ `Notes` de la tâche, et passe la tâche `To Do → To Validate`. **Il ne crée rien dans Wix et n'envoie rien.** Karter compose la campagne dans l'éditeur Wix Email Marketing et l'envoie lui-même, derrière son gate.

## 1. Pourquoi un tracker (contrainte plateforme tranchée le 19.07)
L'**API Wix Email Marketing ne permet PAS de créer ni d'éditer une campagne** : `wix.emailmarketing.api.v1.CampaignService` expose uniquement `Get / List / Delete / Publish / SendTest / PauseScheduling / Reschedule / Reuse / GetAudience / IdentifySenderAddress`. **Aucun `Create`, aucun `Update`.** La composition d'une newsletter reste donc un **geste éditeur manuel** (même mur que Canva pour M3). → Karter a validé l'**Option 1** : l'automate *prépare et suit*, l'humain *compose et envoie*.

## 2. Runner (état ACTIF)
- **ID :** `HnX3llf1rW6qmlPb` · **Nom :** `AFTRSN-NEWSLETTER-TRACKER` · **6 nœuds** · Error Workflow = Sentinel `xzaH0uWy0idKVphF` · scan planifié 1 min.
- **Chaîne :** `Every minute` → `List copies` (HTTP) → `List tasks` (HTTP) → `Should send?` (Code) → `Assemble brief` (Code) → `Fill task + To Validate` (Notion update).

### 2.1 `List copies` (HTTP POST)
`https://api.notion.com/v1/databases/709b816fe2f84dbf90ee597803a11158/query` · cred `notionApi` `sCsj206WVkNxO0Jl` · header `Notion-Version: 2022-06-28` · body :
```json
{"filter":{"property":"Status","select":{"equals":"Approved"}},"page_size":100}
```
→ ne renvoie que les copies **Approved** (= le gate copie, aligné avec la requête).

### 2.2 `List tasks` (HTTP POST)
`https://api.notion.com/v1/databases/cd210f519f9a4d67a50ef6f50e851dc5/query` · même cred/header · body :
```json
{"filter":{"and":[{"property":"Status","select":{"equals":"To Do"}},{"property":"Task","title":{"contains":"Newsletter"}}]},"page_size":100}
```
→ ne renvoie que les tâches Newsletter en `To Do`.

### 2.3 `Should send?` (Code) — parse les tâches, sans relation
Pour chaque page tâche : `title = properties.Task.title[].plain_text`, `status = properties.Status.select.name`, `executor = properties.Executor.select.name`. `parts = title.split(' · ')`. Éligible si `parts.length>=3 && parts[1]==='Newsletter' && status==='To Do' && executor.indexOf('Agent')!==-1`. Émet `{taskPageId, taskName, wave=parts[0], editionName=parts.slice(2).join(' · ').trim()}`.

### 2.4 `Assemble brief` (Code) — matching par TITRE
Lit toutes les copies via `$('List copies').first().json.results`. Indexe chaque copie par clé `editionName||wave` extraits de son titre (`cparts = title.split(' · ')` ; `wave = dernier segment` ; `editionName = cparts.slice(1, -1).join(' · ')` ; **ignore le préfixe « Copy pack »/« Drafts »**) ; exige `Newsletter subject` non vide. Pour chaque tâche, matche par `editionName||wave`. Si aucune copie ⇒ **skip** (la tâche reste To Do). Construit `Notes` :
```
NEWSLETTER — ready to compose & send in Wix (draft, send behind your gate)

SUBJECT:
<Newsletter subject>

PREVIEW:
<Newsletter preview>

BODY:
<Newsletter body>

— Source copy: <copy Title>
```

### 2.5 `Fill task + To Validate` (Notion update, `onError: continueRegularOutput`)
`operation=update`, `pageId` mode url `=https://www.notion.so/{{ $json.taskPageId }}`, propriétés : `Notes|rich_text` = `{{ $json.notesText }}`, `Status|select` = `To Validate`.

## 3. Drainage & idempotence
Source = la **tâche** ; on mute son `Status` To Do→To Validate ; `List tasks` filtre sur `To Do` ⇒ une tâche remplie ne repasse plus (auto-drainage). **Multi-items** : toutes les tâches Newsletter éligibles d'un tick sont traitées. Copie absente ou non-Approved ⇒ la tâche **reste To Do**, réessayée au tick suivant.

## 4. Gate copie = `Approved` (choix Karter 19.07)
La tâche n'est remplie **que si la copie de la vague est `Approved`** dans Content & Social. Karter approuve d'abord la copie (Iris/M2 la dépose en `To Validate`), puis M5 la sert dans la tâche. Double filet : il revoit encore le texte en composant dans Wix.

## 5. Preuve live (19.07) — 3 comportements
1. **Copie Approved** → tâche `To Validate` avec la copie assemblée dans `Notes`. ✅
2. **Copie To Validate (pas Approved)** → tâche reste `To Do` (gate bloque). ✅
3. **Tâche sans copie** → reste `To Do` (garde-fou). ✅ · Multi-items OK. Données de test nettoyées (copie de test archivée, tâche de test remise To Do).

## 6. IDs
Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` · Content & Social DB `709b816fe2f84dbf90ee597803a11158` · cred Notion `sCsj206WVkNxO0Jl` · Sentinel `xzaH0uWy0idKVphF`. La tâche « Newsletter » est posée par M1 (jobs par vague : Copy & captions, Flyer/visual, Wix event page, **Newsletter**, Instagram post + story).

## 7. Suite
M5b / M8 : boucle **stats** newsletter (API Wix `GET /email-marketing/v1/campaigns/statistics` : delivered/opened/clicked/bounced/complained/notSent + landingPage ; perm `Shoutout.Read`) → recap d'édition Notion. Voir POS-GÉNÉRIQUE + mémoire `aftrsn-m5-newsletter-tracker`.
