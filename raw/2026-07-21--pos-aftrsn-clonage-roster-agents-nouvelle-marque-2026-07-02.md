---
type: raw
title: "POS-AFTRSN_Clonage-roster-agents-nouvelle-marque_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_clonage-roster-agents-nouvelle-marque_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# AFTRSN-LUMINA — POS EXACT — Clonage du roster d'agents vers une nouvelle marque (PLAYBOOK v2) — 2026-07-02

Provisionner une nouvelle marque = 1) créer sa banque mémoire (`LUMINA-BRAND-PROVISION`), 2) ingérer son canon, 3) **cloner le roster d'agents par script** (ce POS), qui hérite d'emblée des skills génériques (`lumina`).

## 0. Références

| Élément | Valeur |
|---|---|
| Maestro (source) | `OlrQO21u178SjgBK` |
| Sous-agents spécifiques marque (clonés) | Culture-Steward `vK6B2WuDfbse41Jv`, Experience-Designer `bBu5ycRwHoK7ACdV`, Secretary `FtPFOKvi2t18DFb1`, Marketing `wyUEojwZjSKFRaRL`, Comptable-Finance `F1wsXbsYZdEkwQ3n`, Qualite-Compliance `zbW4p5qsBYhkfBdi` |
| Outils **partagés** (JAMAIS clonés) | Router `Cu8SozYmondKM8RB`, Save Episode/memory-write `zu4jfZbmDz8trQLl` |
| Provisioning marque | `LUMINA-BRAND-PROVISION` `Ul7u33t5VG8MDXWj` |
| Skills (recherche) | Hermes-Exec `Dwv4rcMqNAyQzlrF` → `lumina` + `<marque active>_memory` |

## 1. Principe

Le roster est cloné, pas ré-écrit. Pour chaque marque `<code>` :

1. **Sous-agents** (6) : copie profonde des nœuds (nouveaux `id` uuid), puis deux rebinds :
   - **Knowledge** (`httpRequestTool`) : dans `jsonBody`, `"brand": "aftrsn"` → `"brand": "<code>"` → la mémoire de l'agent est scellée sur la banque `<code>_memory`.
   - **Prompt** (`AI Agent.options.systemMessage`) : on préfixe un bloc *BRAND CONTEXT* (nom, voix/ton, mots interdits tirés de `brand_profiles`) et on remplace les mentions `aftrsn`/`AFTER SUN PEOPLE`.
   - Renommage `<CODE>-<rôle>`, création `POST /rest/workflows`, activation.
2. **Maestro** : copie profonde + prompt adapté, puis **remap** de chaque nœud `toolWorkflow` dont le `workflowId.value` pointe un sous-agent cloné → nouvel id + `cachedResultName`. Router et Save Episode ne sont **pas** remappés (brand-agnostiques : ils dérivent la marque de l'input).

Les **connexions n8n sont par nom de nœud**, pas par id → réattribuer les `id` n'impacte pas le câblage interne.

## 2. Script (à coller dans la console Chrome, session n8n de Karter)

Le script définit `window.cloneRoster(code, profile, {live})`. **Dry-run par défaut** (`live:false`) : valide les transforms sans rien créer. Passer `{live:true}` pour créer + activer réellement.

```js
const hdr = {credentials:'include', headers:{'browser-id': localStorage.getItem('n8n-browserId'), 'content-type':'application/json', 'accept':'application/json'}};
const SUBAGENTS = {
  'vK6B2WuDfbse41Jv':'Culture-Steward','bBu5ycRwHoK7ACdV':'Experience-Designer',
  'FtPFOKvi2t18DFb1':'Secretary','wyUEojwZjSKFRaRL':'Marketing',
  'F1wsXbsYZdEkwQ3n':'Comptable-Finance','zbW4p5qsBYhkfBdi':'Qualite-Compliance'
};
const MAESTRO='OlrQO21u178SjgBK';           // Router Cu8SozYmondKM8RB + Save Episode zu4jfZbmDz8trQLl = PARTAGÉS, non clonés
async function getWF(id){ return (await (await fetch('/rest/workflows/'+id,hdr)).json()).data; }
function adaptPrompt(orig, code, p){
  const header=`BRAND CONTEXT — You operate exclusively for the brand "${p.brand_name}" (code: ${code}). Voice & tone: ${p.tone_voice}. Forbidden words: ${(p.forbidden_words||[]).join(', ')||'none'}. Your Knowledge tool is scoped to this brand's memory bank only.\n\n`;
  return header+(orig||'').replace(/aftrsn/gi, code).replace(/AFTER SUN PEOPLE/gi, p.brand_name);
}
function freshIds(nodes){ nodes.forEach(n=>{ n.id = crypto.randomUUID(); }); return nodes; }
function buildSubClone(src, code, p){
  const w=JSON.parse(JSON.stringify(src)); const nodes=freshIds(w.nodes);
  const kn=nodes.find(n=>/knowledge/i.test(n.name) && /httpRequestTool/i.test(n.type));
  if(kn && kn.parameters.jsonBody){ kn.parameters.jsonBody = kn.parameters.jsonBody.replace(/"brand"\s*:\s*"[^"]*"/g,'"brand": "'+code+'"'); }
  if(kn && kn.parameters.toolDescription){ kn.parameters.toolDescription = kn.parameters.toolDescription.replace(/aftrsn/gi,code); }
  const ag=nodes.find(n=>/agent/i.test(n.type) && n.parameters && n.parameters.options && 'systemMessage' in n.parameters.options);
  if(ag){ ag.parameters.options.systemMessage = adaptPrompt(ag.parameters.options.systemMessage, code, p); }
  const role=SUBAGENTS[src.id];
  return {name:code.toUpperCase()+'-'+role, nodes, connections:w.connections, settings:w.settings||{}, role, srcId:src.id};
}
function buildMaestroClone(src, code, p, idMap){
  const w=JSON.parse(JSON.stringify(src)); const nodes=freshIds(w.nodes);
  const ag=nodes.find(n=>/agent/i.test(n.type) && n.parameters && n.parameters.options && 'systemMessage' in n.parameters.options);
  if(ag){ ag.parameters.options.systemMessage = adaptPrompt(ag.parameters.options.systemMessage, code, p); }
  let remapped=[];
  nodes.filter(n=>n.type==='@n8n/n8n-nodes-langchain.toolWorkflow').forEach(n=>{
    const wf=n.parameters.workflowId; const oldId=(wf&&wf.value)||wf;
    if(idMap[oldId]){ n.parameters.workflowId={__rl:true,mode:'list',value:idMap[oldId].id,cachedResultName:idMap[oldId].name}; remapped.push(n.name+'→'+idMap[oldId].name); }
  });
  return {name:code.toUpperCase()+'-Maestro', nodes, connections:w.connections, settings:w.settings||{}, remapped};
}
async function cloneRoster(code, profile, opts={}){
  const live=!!opts.live; const report={code, live, subagents:[], maestro:null, errors:[]}; const idMap={};
  for(const id of Object.keys(SUBAGENTS)){
    const src=await getWF(id); const c=buildSubClone(src, code, profile);
    const kn=c.nodes.find(n=>/knowledge/i.test(n.name));
    const brandOk=kn? kn.parameters.jsonBody.includes('"brand": "'+code+'"') : false;
    let newId='(dry)';
    if(live){
      const r=await fetch('/rest/workflows',{method:'POST',...hdr,body:JSON.stringify({name:c.name,nodes:c.nodes,connections:c.connections,settings:c.settings})});
      const j=await r.json(); if(r.ok){ newId=j.data.id; await fetch('/rest/workflows/'+newId+'/activate',{method:'POST',...hdr,body:JSON.stringify({versionId:j.data.versionId})}); } else { report.errors.push('sub '+c.name+' HTTP '+r.status); }
    }
    idMap[id]={id:newId,name:c.name};
    report.subagents.push({name:c.name, brandRebound:brandOk, nodes:c.nodes.length, newId});
  }
  const msrc=await getWF(MAESTRO); const mc=buildMaestroClone(msrc, code, profile, idMap);
  let mId='(dry)';
  if(live){
    const r=await fetch('/rest/workflows',{method:'POST',...hdr,body:JSON.stringify({name:mc.name,nodes:mc.nodes,connections:mc.connections,settings:mc.settings})});
    const j=await r.json(); if(r.ok){ mId=j.data.id; await fetch('/rest/workflows/'+mId+'/activate',{method:'POST',...hdr,body:JSON.stringify({versionId:j.data.versionId})}); } else { report.errors.push('maestro HTTP '+r.status); }
  }
  report.maestro={name:mc.name, remapped:mc.remapped, newId:mId};
  return report;
}
window.cloneRoster=cloneRoster;

// Exemple (dry-run) :
await cloneRoster('demo2',{brand_name:'Demo Brand Two',tone_voice:'chaleureux, direct',forbidden_words:['cheap','spam']},{live:false});
// Réel : ... {live:true}
```

## 3. Procédure complète d'onboarding d'une marque

1. `LUMINA-BRAND-PROVISION` avec `{code, brand_name, tone_voice, forbidden_words, kb_ref}` → crée `<code>_memory` + INSERT `memory_registry` + `brand_profiles`.
2. Ingérer le **canon** de la marque (`collection='canon'`) via l'ingestion multi-format.
3. **Dry-run** `cloneRoster(code, profile, {live:false})` → vérifier `brandRebound:true` partout + les 6 remaps Maestro.
4. **Réel** `cloneRoster(code, profile, {live:true})` → 6 sous-agents + Maestro créés/activés, préfixés `<CODE>-`.
5. Tester : ouvrir le chat de `<CODE>-Maestro` → une question métier → vérifier délégation + réponse marquée à la bonne voix, Knowledge scellé sur `<code>_memory`.

## 4. Pièges figés

1. **Ne jamais cloner les outils partagés** (Router, Save Episode) : ils sont brand-agnostiques et dérivent la marque de l'input. Les cloner créerait des doublons inutiles et casserait le routage central.
2. **Rebind Knowledge = source de l'isolation mémoire** : si `"brand"` n'est pas ré-écrit, le clone lit la banque d'AFTRSN. Toujours vérifier `brandRebound:true` avant `live:true`.
3. **Connexions par nom** : garder les noms de nœuds identiques ; seul l'`id` change. Ne pas renommer les nœuds internes.
4. **Prompts** : le bloc BRAND CONTEXT + le `brand_profile` portent la voix ; les skills génériques (`lumina`) sont hérités automatiquement, les skills spécifiques marque s'accumulent dans `<code>_memory` (collection `skills`) et remontent via Hermes-Exec (recherche `lumina` + `<marque active>`).
5. **Nettoyage d'un clone de test** : suppression de workflows = action destructive → à faire par Karter dans n8n (l'assistant ne supprime pas).
6. **Activation** exige des credentials valides sur tous les nœuds (Postgres, OpenRouter, OpenAI embed) — hérités de la source, donc OK.

---
*POS EXACT — 2026-07-02, Claude (Cowork). PLAYBOOK v2. Version générique : POS-GENERIQUE clonage roster multi-marques. Validé en dry-run (6 sous-agents + Maestro, brand rebound OK, 6 remaps, Router/Save Episode préservés).*
