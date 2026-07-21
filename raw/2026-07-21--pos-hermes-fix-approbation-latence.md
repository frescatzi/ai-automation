---
type: raw
title: "POS-HERMES_Fix-Approbation-Latence"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-hermes_fix-approbation-latence.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# 🛠️ MARCHE À SUIVRE — Fix Hermès « pas de réponse / latence » (Phase A)
**Version :** v1.0 · **Date :** 2026-07-10 · **Projet :** LUMINA OS · **Workflow cible :** `LUMINA-Hermes-Exec` (`Dwv4rcMqNAyQzlrF`), node **`Hermes Exec`** (Code)
**Statut :** prêt à appliquer (Phase A — MVP). **n8n fait foi.** À appliquer par Karter dans l'éditeur n8n (le node Code se colle à la main).

---

## 1. Cause racine (diagnostiquée en live 09→10-07)

Pour une requête web/shell, Hermès veut exécuter un outil (`terminal`, ex. `curl`). Son scanner de sécurité (tirith) **met l'exécution en pause sur une demande d'approbation**. Dans l'UI, Karter clique « Allow » → réponse en ~15 s. **Côté n8n, personne n'approuve** → Hermès reste bloqué, `export` renvoie `messages: []` + `active_stream_id` actif indéfiniment → la boucle de poll épuise ses 42 s → fallback `« pas de réponse dans le délai »` (avec `outcome: success`).

Preuve dans les données (`tool_calls` d'une session bloquée) : `"BLOCKED: Command timed out without user response. The user has NOT consented to this action."`

Deux défauts cumulés dans le node actuel :
1. **Aucune gestion de l'approbation** → blocage sur tout outil « gated » (web, shell).
2. **Détection de fin fragile** : la boucle exige « 2 exports consécutifs identiques », ce qui rate les réponses qui streament.

## 2. Mécanique API confirmée (hermes.aftersunpeople.com)

- `POST /api/session/new` → `{ session:{ session_id } }`
- `POST /api/chat/start` `{ session_id, message }` → lance la génération (flux SSE côté UI)
- **Approbation (HTTP, PAS WebSocket) :** `POST /api/approval/respond` body `{ "session_id", "choice", "approval_id": null }`. `choice` ∈ `once` | `session` | `always` | `deny`. `approval_id:null` = l'approbation en cours. *(Le mode `yolo` existe mais sa bascule passe par WebSocket → inutilisable depuis n8n. D'où le choix de l'endpoint d'approbation.)*
- **Fin de génération :** `GET /api/session/export?session_id=…` → terminé quand `active_stream_id === null` ET `pending_user_message == null`, avec au moins un message `role:"assistant"`.
- **Skills/outils natifs utilisés :** `export.tool_calls[]` → `{ name, args.command, snippet, tid, assistant_msg_idx }`.

## 3. Le correctif — nouveau code du node `Hermes Exec`

> Ne change QUE le bloc d'exécution (login → poll). Le début du node (CANON-INJECT, construction de `message`, `skills`/`skillText`, `slugs`) reste **inchangé**. Remplace tout ce qui suit la ligne `const login=$input.first().json;` jusqu'au `return` final par :

```js
const login=$input.first().json;
const sc=login.headers&&login.headers['set-cookie'];
const cookie=Array.isArray(sc)?sc.map(c=>c.split(';')[0]).join('; '):String(sc||'').split(';')[0];
const H={Cookie:cookie,'Content-Type':'application/json'};
const sleep=ms=>new Promise(r=>setTimeout(r,ms));
let outcome='success', text='', usedTools=[];
try{
  if(!cookie) throw new Error('no_cookie');
  const ns=await this.helpers.httpRequest({method:'POST',url:base+'/api/session/new',headers:H,body:{},json:true});
  const sid=ns.session.session_id;
  await this.helpers.httpRequest({method:'POST',url:base+'/api/chat/start',headers:H,body:{session_id:sid,message:skillText+message},json:true});
  let done=false, tries=0; const MAX=40; // 40 × 1,5s = 60s de fenêtre
  while(tries<MAX){ await sleep(1500); tries++;
    // (A) PHASE A : auto-approuve toute demande d'approbation en attente (no-op si rien en attente)
    try{ await this.helpers.httpRequest({method:'POST',url:base+'/api/approval/respond',headers:H,body:{session_id:sid,choice:'once',approval_id:null},json:true}); }catch(e){}
    // (B) fin de génération ? signal fiable = active_stream_id null + plus de pending + message assistant présent
    let ex; try{ ex=await this.helpers.httpRequest({method:'GET',url:base+'/api/session/export',qs:{session_id:sid},headers:{Cookie:cookie},json:true}); }catch(e){ continue; }
    const asst=((ex&&ex.messages)||[]).filter(m=>m.role==='assistant');
    if(ex && ex.active_stream_id===null && !ex.pending_user_message && asst.length){
      const c=asst[asst.length-1].content; text=(typeof c==='string')?c:JSON.stringify(c);
      usedTools=((ex.tool_calls)||[]).map(t=>t.name);
      done=true; break;
    }
  }
  if(!done){ outcome='timeout'; text=text||'Hermes: pas de réponse dans le délai.'; }
}catch(e){ outcome='fail'; text='Hermes erreur: '+String(e.message||e).slice(0,120); }
return [{json:{text, outcome, slugs, usedSkills:skills.map(s=>s.title), usedTools}}];
```

**Ce qui change vs l'ancien :**
- Ajout de l'auto-approbation HTTP à chaque tour de boucle (le déblocage).
- Détection de fin par `active_stream_id===null` (au lieu de « 2 polls identiques »).
- Poll ramené à 1,5 s, fenêtre portée à 60 s.
- Nouveau champ de sortie `usedTools` (= `tool_calls[].name`, base de l'observatoire skills).

## 4. Test de validation (après collage)

1. Depuis Lyra (Telegram) : « Quel temps fait-il à Lugano aujourd'hui ? ».
2. Attendu : réponse météo réelle livrée dans Telegram en **~15–25 s** (plus de « pas de réponse dans le délai »).
3. Vérifier une requête d'exécution pure (sans web) : doit rester rapide.
4. Dans l'exécution n8n, contrôler le champ `usedTools` (doit lister `terminal`, etc.).

**Rollback :** restaurer la version précédente du node depuis l'historique de versions n8n si régression.

## 5. Suite — Phase B (supervision Maestro) et chantiers 2 & 3

- **Phase B (gouvernance) :** remplacer l'auto-approbation aveugle (`choice:'once'` systématique) par une **politique** : lire l'approbation en attente via `GET /api/approval/stream` (SSE), auto-approuver le non-critique, `choice:'deny'` + **escalade Telegram vers Karter** pour le critique (dépense / envoi / publication / suppression).
- **Chantier 2 (Hermès-first) :** alléger le pré-mâchage (embeddings + skills pgvector + CANON-INJECT) → appeler Hermès quasi nu par défaut, mémoire en **fallback** (échec ou sortie jugée « pas bon »).
- **Chantier 3 (observatoire) :** journaliser `usedTools` par demande ; paliers d'usage (10 → 20 → 50) → un skill « remonte en surface » comme candidat à partager avec les autres agents et à injecter en base, **sous validation Maestro** (branché sur `Save Lesson` / `Save Episode` de `AFTRSN-Maestro` `OlrQO21u178SjgBK`).

---
*Marche à suivre Hermès Fix Phase A v1.0 — 2026-07-10. Diagnostic prouvé en live (trace réseau UI + export session). Phase A = déblocage ; Phases B / chantiers 2-3 = refonte Hermès-first + supervision Maestro.*
