---
type: raw
title: "POS-EXACT_AutoEvent-P3_M1b-Iris-auto-declenchement_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m1b-iris-auto-declenchement_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Phase 3 · M1b · Iris : auto-déclenchement de l'orchestrateur

**Date :** 17.07.2026 · **Marque :** AFTRSN · **Espace :** n8n LUMINA OS + Notion Editions (AFTER SUN PEOPLE - HQ)
**Statut :** ✅ TERMINÉ, TESTÉ & ACTIVÉ (17.07.2026). Évolution de M1 : l'orchestrateur devient autonome. L'agent qui prépare le contenu événementiel s'appelle désormais **Iris**.
**Workflow :** `AFTRSN-IRIS-EVENT-AGENT` = `git92S6gDU1hGZ2b` (ex `AFTRSN-EVENT-ORCHESTRATOR`, renommé pour acter l'agent Iris ; 9 nodes, **actif**, poll 10 min).

---

## 1. Objectif
Rendre M1 autonome : au lieu d'un clic manuel sur une édition pilote, Iris écoute la base Editions et génère automatiquement la vague de contenu quand le **line-up** change, sans jamais rejouer inutilement.

## 2. Déclenchement
- Suppression du Manual Trigger et du garde-fou pilote (plus d'ID Mini Café en dur).
- **2 Notion Triggers** (typeVersion 1, credential `AFTRSN-n8n-Backstage` `sCsj206WVkNxO0Jl`) sur la base Editions **DB `dad7f826891f4c2980e4183c59bac8b0`** : event `pageAddedToDatabase` (création) et `pagedUpdatedInDatabase` (modification), `simple:true`, `pollTimes` = toutes les 10 min.
- Les deux triggers convergent vers **Read the edition** (Notion get, `pageId` = `=https://www.notion.so/{{ $json.id }}`). Périmètre : toutes les éditions ; le trigger ne voit que les changements postérieurs à l'activation (pas de rattrapage historique).

## 3. Réagir UNIQUEMENT au line-up (empreinte)
Le trigger voit toute la page ; le filtrage se fait dans le flux (node **Build the brief**) :
- **Empreinte** = tokens du line-up normalisés + triés + joints par `|` : notes `Lineup notes` (split `, ; \n`, trim, lowercase) + IDs de la relation `Lineup` (sans tirets, lowercase). Vide si pas de line-up.
- **Mémoire** : champ texte caché **`Iris line-up seen`** sur Editions (ajouté). Lu via `property_iris_line_up_seen` (fallback : scan des clés contenant « iris »).
- Empreinte courante == mémorisée → 0 tâche (date/lieu/budget ignorés). Différente ET line-up non vide → nouvelle vague.

## 4. Modèle des vagues (déterminé depuis la checklist existante de l'édition)
`Read existing checklist` (getAll Tasks, returnAll) → on ne garde que les tâches dont la relation `Event` pointe l'édition courante, puis :
- `hasAnnounceWave`, `hasRevealWave`, `updateCount` (max N des « Line-up update N »).
- **Auto** : pas de line-up + pas d'annonce → **Announce**. Line-up + empreinte changée + pas de reveal → **Line-up reveal**. Line-up + empreinte changée + reveal déjà là → **Line-up update (updateCount+1)**. Empreinte inchangée → rien.
- **Content Phase = Announce** + line-up présent → **verrou** (0 tâche). Announce + pas de line-up → Announce (si pas déjà fait).
- **Content Phase = Line-up reveal** → reveal (ou update N si reveal déjà là).

## 5. Tâches (5 par vague) et fin de run
Catalogue unique de 5 jobs : `Copy & captions`, `Flyer/visual`, `Wix event page`, `Newsletter`, `Instagram post + story`. Nommage `{Vague} · {job} · {édition}` (séparateur « · »). Status `To Do`, Executor `🤖 Agent`, Category `Brand & Community`, relation `Event` = `={{ [$json.eventId] }}`. Dédoublonnage de sécurité par nom exact.
Fin de run : **Keep one item** (n8n Limit, maxItems 1) puis **Write line-up fingerprint** (Notion update, `Iris line-up seen|rich_text` = empreinte courante). L'écriture n'a lieu QUE si une vague a été créée (chaînée après Generate).

## 6. Tests réels (Fetch Test Event, workflow encore inactif)
- **#14287** : Mini Café, line-up présent (relation + notes), `Iris line-up seen` vide, reveal déjà là → **Line-up update 1** (5 tâches créées) + empreinte écrite. ✅
- **#14290** : re-run sans changement → `fingerprint == stored` → `waves:[]` → **0 tâche, 0 écriture** (Generate/Write non exécutés). ✅ idempotence + boucle bornée.
Puis **activation** (actif, poll 10 min).

## 7. Limites assumées (MVP)
- **Auto-réveil borné** : écrire l'empreinte est une modif de page → le trigger « Updated » se re-réveille une fois, voit empreinte identique → 0 tâche, 0 écriture → s'arrête. Un tour à vide.
- **Suppression de page** non détectée (sans impact, Iris n'ajoute que des tâches).
- **Réaction « dans le vide »** (ex. budget modifié) : run mais empreinte identique → 0 tâche.
- **Premier contact** sur une édition ayant déjà des tâches reveal mais `Iris line-up seen` vide → une « Line-up update 1 » (cas du pilote Mini Café au test). Sans effet sur les éditions futures (partent propres).

## 8. IDs
Workflow `git92S6gDU1hGZ2b` · Editions DB `dad7f826891f4c2980e4183c59bac8b0` / ds `61418ed4-fb84-48fc-b372-247fbabe0c33` · Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` · cred Notion `sCsj206WVkNxO0Jl` · Sentinel `xzaH0uWy0idKVphF`. Champs Editions : `Content Phase`, `Next content wave`, **`Iris line-up seen`** (nouveau). Mode d'emploi utilisateur : « 🎬 Iris · Event Content Checklist » (wiki Knowledge `3a024f2f-ef0c-819e-8a8c-d5dc126ba8a9`).

---

## Difficultés rencontrées
1. Trigger Notion non déclenchable par API (comme le Manual Trigger) : test via « Fetch Test Event » dans l'éditeur.
2. Distinguer « création sans line-up » (Announce) d'une simple non-modif : l'empreinte seule ne suffit pas (vide == vide).
3. Risque de boucle : écrire l'empreinte re-déclenche le trigger « Updated ».
4. Nommer les vagues successives sans doublon ni réouverture.

## Solutions implémentées
1. Test éditeur + lecture d'exécution par API (`n8n_executions get`), workflow gardé inactif jusqu'à validation.
2. Décision de vague fondée sur la **checklist existante** de l'édition (hasAnnounceWave / hasRevealWave / updateCount), pas seulement sur l'empreinte.
3. Écriture de l'empreinte **uniquement après création d'une vague** (Keep one item après Generate) → au pire un seul réveil à vide, borné.
4. Vagues nommées « Line-up update N » (N = updateCount+1) → uniques, 5 tâches fraîches, pas de réouverture.

## Lessons learned
1. **Un Notion Trigger ne surveille pas un champ précis** : il voit toute la page. Le filtrage « ne réagir qu'à X » se fait dans le flux via une **empreinte** de X + un état mémorisé.
2. **L'idempotence d'un automate déclenché par sa propre écriture** exige de n'écrire que quand on agit, et de comparer l'état avant d'agir → boucle bornée à un tour.
3. **La source de vérité pour « quelle vague » est la checklist déjà posée**, pas seulement l'empreinte : ça survit aux relances et aux redémarrages.
4. **Empreinte robuste = relation + texte** normalisés/triés : capte le line-up quel que soit le mode de saisie (DJ en relation ou en notes).
5. Séparateur « · » partout (jamais « — ») ; le « — » du nom d'édition reste la donnée de Karter.
