---
type: raw
title: "POS-GENERIQUE_Onboarding-nouvelle-marque_roster-clone_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_onboarding-nouvelle-marque_roster-clone_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: sunkids
immutable: true
---

# LUMINA — MARCHE À SUIVRE — Onboarding d'une nouvelle marque (provisioning → clone du roster → test) — 2026-07-02

Procédure de bout en bout pour brancher une **nouvelle marque** sur LUMINA : créer sa mémoire, ingérer son canon, cloner son roster d'agents, le ranger au standard, et **vérifier que le Maestro cloné délègue bien**. À suivre dans l'ordre ; chaque étape a sa vérification.

> Toutes les commandes API s'exécutent via `fetch` dans la **session Chrome n8n de Karter** (header `browser-id` = `localStorage['n8n-browserId']`). Rien de destructif n'est fait par l'assistant : toute suppression éventuelle reste à la main de Karter.

## 0. Prérequis — réunir le profil de marque

| Champ | Exemple | Sert à |
|---|---|---|
| `code` | `sunkids` | nom de la banque `sunkids_memory`, préfixe workflows `SUNKIDS-`, tag marque |
| `brand_name` | `Sun Kids` | identité dans les prompts |
| `tone_voice` | `ludique, rassurant, parental` | voix des agents |
| `forbidden_words` | `["cheap","danger"]` | garde-fous de voix |
| `kb_ref` | id dossier Drive du canon | source d'ingestion |

Règle `code` : minuscules, `[a-z0-9_]` uniquement (sanitizé partout).

## 1. Provisionner la banque mémoire

Exécuter le workflow **`LUMINA-BRAND-PROVISION`** (`Ul7u33t5VG8MDXWj`) avec l'input :

```json
{ "code":"sunkids", "brand_name":"Sun Kids", "tone_voice":"ludique, rassurant, parental",
  "forbidden_words":["cheap","danger"], "kb_ref":"<idDossierDriveCanon>" }
```

**Crée** : table `sunkids_memory` (schéma + index HNSW cosinus), ligne `memory_registry`, ligne `brand_profiles`.
**Vérifier** : `SELECT * FROM memory_registry WHERE code='sunkids';` (1 ligne) et la table `sunkids_memory` existe (0 ligne, normal).

## 2. Créer le squelette de dossiers + le tag marque (standard process-flow)

Voir POS *Classification workflows n8n*. Via API :

```js
const hdr={credentials:'include',headers:{'browser-id':localStorage.getItem('n8n-browserId'),'content-type':'application/json','accept':'application/json'}};
const PID='Tn1aNTcuxmqqHnKU';
// a) tag marque
const brandTag=(await (await fetch('/rest/tags',{method:'POST',...hdr,body:JSON.stringify({name:'🧒 SUNKIDS'})})).json()).data.id;
// b) racine marque + dossiers-étapes
const root=(await (await fetch(`/rest/projects/${PID}/folders`,{method:'POST',...hdr,body:JSON.stringify({name:'03-SUNKIDS'})})).json()).data.id;
const F={};
for(const [k,n] of [['INTAKE','01-INTAKE'],['MEM','02-MEMORY'],['BRAIN','03-BRAIN'],['AUTO','04-AUTOMATION'],['CONNECT','05-CONNECT'],['UTIL','08-UTIL']]){
  F[k]=(await (await fetch(`/rest/projects/${PID}/folders`,{method:'POST',...hdr,body:JSON.stringify({name:'SUNKIDS-'+n,parentFolderId:root})})).json()).data.id;
}
// sous-dossier Sub-Agents dans BRAIN
F.SUB=(await (await fetch(`/rest/projects/${PID}/folders`,{method:'POST',...hdr,body:JSON.stringify({name:'Sub-Agents',parentFolderId:F.BRAIN})})).json()).data.id;
```

Noter `brandTag`, `F.BRAIN`, `F.SUB`, `F.MEM` pour les étapes 5-6.

## 3. Ingérer le canon de la marque

Alimenter `collection='canon'` dans `sunkids_memory` :
- **Dossier Drive** → cloner/appeler l'ingestion récursive (patron `AFTRSN-MEMORY-INGESTION/DRIVE-RECURSIVE`) avec `{brand:'sunkids', folderId:kb_ref}` — **PDF + markdown uniquement**.
- **Textes ponctuels** → `LUMINA-MEMORY-INGEST-TEXT` (`ugINEyyBKRLcDpH2`) avec `{brand:'sunkids', title, content, collection:'canon', source_ref}`.

**Vérifier** : `SELECT count(*) FROM sunkids_memory WHERE collection='canon';` > 0, et une recherche sur un fait connu remonte avec un bon score.

## 4. Dry-run du clone (valider sans rien créer)

Coller le script `cloneRoster` (POS *Clonage roster*), puis :

```js
const profile={brand_name:'Sun Kids',tone_voice:'ludique, rassurant, parental',forbidden_words:['cheap','danger']};
await cloneRoster('sunkids', profile, {live:false});
```

**Vérifier** dans le rapport : `brandRebound:true` pour les **6 sous-agents** + **6 remaps** Maestro (Router & Save Episode **non** remappés). Si un `brandRebound:false` → stop, inspecter le nœud Knowledge source avant d'aller en live.

## 5. Clone réel (créer + activer)

```js
const rep=await cloneRoster('sunkids', profile, {live:true});
// rep.subagents[].newId + rep.maestro.newId = ids créés
```

**Crée + active** : 6 sous-agents `SUNKIDS-<rôle>` + `SUNKIDS-Maestro`, mémoire Knowledge scellée sur `sunkids_memory`, prompts à la voix Sun Kids. **Vérifier** : `rep.errors` vide ; les 7 workflows apparaissent (Published).

## 6. Ranger + taguer les nouveaux workflows

Déplacer les clones dans le squelette + appliquer les 4 axes (Domaine 🤖 AI-AGENTS, Statut 🟢 PROD, Marque = `brandTag`) :

```js
const AG='PLsFqlaD1O3kVGZb', PROD='sMHN7rMS8y2Tuqc8';
// Maestro -> 03-BRAIN ; sous-agents -> Sub-Agents
await fetch('/rest/workflows/'+rep.maestro.newId,{method:'PATCH',...hdr,body:JSON.stringify({parentFolderId:F.BRAIN, tags:[AG,PROD,brandTag]})});
for(const s of rep.subagents){ await fetch('/rest/workflows/'+s.newId,{method:'PATCH',...hdr,body:JSON.stringify({parentFolderId:F.SUB, tags:[AG,PROD,brandTag]})}); }
```

Les ingestions de la marque (étape 3) → `SUNKIDS-02-MEMORY` (`F.MEM`), tags `🧠 MEMORY + déclencheur + brandTag`.
**Vérifier** : aucun clone à la racine, tous tagués (mêmes contrôles que le POS classification).

## 7. Tester la délégation du Maestro cloné

But : prouver que le hub délègue au bon sous-agent, à la bonne voix, sur la bonne mémoire.

1. Ouvrir le chat de **`SUNKIDS-Maestro`** (ou lancer une exécution via API avec `pinData` sur le chatTrigger).
2. Poser une question métier qui force une délégation, ex. : *« Demande au comptable une estimation budgétaire rapide pour un événement de 200 personnes. »*
3. **Vérifier** dans l'exécution :
   - un nœud `Call 'SUNKIDS-Comptable-Finance'` (ou autre sous-agent) a été appelé ;
   - la réponse est chiffrée / cohérente et **à la voix Sun Kids** ;
   - aucun mot interdit ;
   - si la question touche au canon, l'info vient bien de `sunkids_memory` (pas d'AFTRSN).
4. Refaire un test ciblant un **2ᵉ sous-agent** (ex. expérience/événement) pour confirmer le routage multiple.

Méthode API (rappel) : `POST /rest/workflows/<maestroId>/run` avec `{workflowData:<wf>, triggerToStartFrom:{name:'When chat message received'}}` + `pinData`, puis lire `/rest/executions/<id>` (status `success`, nœud `Call '…'` présent).

## 8. Clôture & filet de sécurité

- **Skills** : la marque hérite d'emblée des skills génériques (`lumina`) ; ses skills propres s'accumuleront dans `sunkids_memory` (collection `skills`) et remonteront via Hermes-Exec (recherche `lumina` + marque active, seuil de similarité 0.35).
- **Router / mémoire / Hermes** = **partagés**, rien à cloner.
- **Rollback** (si clone raté) : désactiver puis **supprimer** les workflows `SUNKIDS-*` et `DROP TABLE sunkids_memory` + lignes registry/profiles → **opérations destructives réservées à Karter** (l'assistant ne supprime pas).

## Checklist express

- [ ] Profil réuni (code/brand_name/tone/forbidden/kb_ref)
- [ ] `LUMINA-BRAND-PROVISION` OK (registry + table)
- [ ] Squelette dossiers + tag marque créés
- [ ] Canon ingéré (count > 0, recherche OK)
- [ ] Dry-run : brandRebound ×6 + remaps ×6
- [ ] Clone live : 7 workflows, `errors` vide
- [ ] Rangés (03-BRAIN / Sub-Agents) + tagués (4 axes)
- [ ] Test délégation : bon sous-agent + bonne voix + bonne mémoire (×2)

---
*MARCHE À SUIVRE — 2026-07-02, Claude (Cowork). Réunit : POS Clonage-roster (cloneRoster), POS Classification process-flow, PLAYBOOK v1 (provisioning). Exemple fil rouge : marque fictive « sunkids ». Point 16 du plan LUMINA AI OS.*
