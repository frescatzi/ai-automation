---
type: raw
title: "POS-EXACT_AutoEvent-P3_M1-EDITION-WATCHER_a-jour_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_m1-edition-watcher_a-jour_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT (à jour) · Phase 3 · M1 · `AFTRSN-EDITION-WATCHER`

**Date :** 19.07.2026 (version à jour, relue depuis le workflow live) · **Marque :** AFTER SUN PEOPLE (AFTRSN) · **Espaces :** n8n LUMINA OS + Notion « The Backstage ».
**Statut :** ✅ EN SERVICE. Workflow `AFTRSN-EDITION-WATCHER` = `git92S6gDU1hGZ2b` (10 nœuds, **actif**, **scan planifié 1 min**). Error Workflow = Sentinel `xzaH0uWy0idKVphF`.

> Cette version reflète l'état RÉEL du workflow (refactor scan du 18.07 + gel des éditions + garde `hasDateVenue`), pas la première ébauche à triggers. Terminologie fixée : c'est un **workflow déterministe** (« watcher »), **pas un agent IA**. « Iris » = l'agent Channel Content (`AFTRSN-IRIS-CHANNEL-CONTENT`, `MWUwLi3vu2Ws65lL`), qui rédige la copy et qui est appelé plus loin par le Copy Runner (M2).

---

## 1. Rôle
Le watcher **surveille la base Editions** et, quand une édition passe des jalons (date + lieu connus, puis line-up posé ou modifié), il **pose automatiquement la checklist de contenu** (une « vague » de 5 tâches) dans la base Tasks, en `To Do` / `🤖 Agent`. Il ne rédige rien et ne publie rien : il **pose le travail à faire**, que les runners suivants (M2 à M6) consomment. Aucun LLM, purement déterministe.

## 2. Déclenchement : scan planifié 1 min (pas de trigger Notion)
- Nœud **`Every minute`** (`scheduleTrigger`, intervalle 1 min).
- **Pourquoi le scan et pas le Notion Trigger « Updated » :** le trigger Notion n'est pas déclenchable par API (test impossible proprement), voit toute la page (pas un champ), et posait des soucis de drainage multi-items. Le scan **reprend à chaque tick toutes les éditions éligibles**, quel que soit le moment du changement. (Refactor 18.07.)

## 3. Filtre de gel des éditions
- **`List editions`** (Notion `getAll`, Editions DB `dad7f826891f4c2980e4183c59bac8b0`, returnAll).
- **`Active editions`** (Code) : **écarte** les éditions dont le `Status` ∈ {`Live`, `Cancelled`, `Done`} ; ne laisse passer que les éditions à venir (Planning/Confirmed) sous la forme `{ id }`. Ainsi le watcher ne re-traite jamais une édition passée, annulée ou en cours.

## 4. Boucle multi-items
- **`Loop editions`** (`splitInBatches`, batchSize 1) : traite **une édition par tick**, et **reboucle** en fin de run (drainage 1/tick, évite d'affamer les vagues au-delà de la première).

## 5. Lecture + empreinte du line-up
- **`Read the edition`** (Notion `get`, `pageId = https://www.notion.so/{{ $json.id }}`).
- **`Read existing checklist`** (Notion `getAll` Tasks `cd210f519f9a4d67a50ef6f50e851dc5`, returnAll, `alwaysOutputData`) : sert à savoir quelles vagues existent déjà pour l'édition.
- **`Build the brief`** (Code) calcule :
  - `hasDateVenue` = `Date.start` présent **ET** relation `Venue` non vide.
  - **empreinte du line-up** = tokens normalisés/triés/joints `|` : `Lineup notes` (split `, ; \n`, trim, lowercase) + IDs de la relation `Lineup` (sans tirets, lowercase). Vide si pas de line-up.
  - `stored` = champ `Line-up seen` (`property_line_up_seen`, fallback scan clé « seen »).
  - `phaseField` = champ `Content Phase`.

## 6. Modèle des vagues (logique exacte du code)
Décision fondée sur la checklist existante (`hasAnnounceWave`, `hasRevealWave`, `updateCount`) + `hasDateVenue` + empreinte :
- **Announce ne se déclenche que si `hasDateVenue`** (date ET lieu connus). Pas de date+venue → pas d'annonce.
- **Content Phase = `Announce`** : line-up présent → **verrou** (0 tâche, `blockedReason`) ; sinon, si pas d'Announce déjà posée et `hasDateVenue` → **Announce**.
- **Content Phase = `Line-up reveal`** : `Line-up reveal` si pas encore posée, sinon `Line-up update N`.
- **Auto (Content Phase vide)** :
  - pas de line-up : si pas d'Announce et `hasDateVenue` → **Announce**.
  - line-up présent et empreinte **inchangée** → rien.
  - line-up présent et empreinte **changée**, pas de reveal : si pas d'Announce et `hasDateVenue` → **[Announce, Line-up reveal]** (les deux vagues d'un coup) ; sinon → **[Line-up reveal]**.
  - line-up présent et empreinte changée, reveal déjà là → **[Line-up update (updateCount+1)]**.

## 7. Tâches créées
- **`Plan production tasks`** (Code) : catalogue de 5 jobs (`Copy & captions`, `Flyer/visual`, `Wix event page`, `Newsletter`, `Instagram post + story`) × chaque vague ciblée ; nommage `{Vague} · {job} · {édition}` ; **dédoublonnage** par nom exact contre la checklist existante de l'édition.
- **`Generate production checklist`** (Notion `create`, Tasks DB) : `Task` = nom, `Status` = `To Do`, `Executor` = `🤖 Agent`, `Category` = `Brand & Community`, relation `Event` = `[$json.eventId]` (tableau). Jamais « Done ».

## 8. Fin de run
- **`Write line-up fingerprint`** (Notion `update`, `Line-up seen|rich_text` = empreinte courante, `onError: continueRegularOutput`) part de `Build the brief` puis **reboucle sur `Loop editions`**. L'écriture de l'empreinte évite de rejouer une vague sur un line-up déjà vu.

## 9. Champs Notion (base Editions, ds `61418ed4-fb84-48fc-b372-247fbabe0c33`)
- `Content Phase` (select : `Announce` / `Line-up reveal` ; vide = Auto) — **lu** par le watcher.
- `Line-up seen` (texte caché, mémoire de l'empreinte) — **lu et écrit** par le watcher.
- `Next content wave` (formule d'affichage de parité) — non lu par le workflow, aide visuelle sur la fiche.

## 10. Place dans le moteur (M1 → M6)
Le watcher (M1) pose les tâches. Elles sont ensuite consommées, en **scan 1 min** chacun, par : **M2 Copy Runner** (`XQYvzTzFK2NLjDZz`, appelle l'agent Iris/Channel Content, écrit dans les bases Content en `To Validate`), **M3 Visual Tracker** (`sJQ9Usa18kceEjlt`), **M4 Wix Event Runner** (`6XVotFtRAZ7BmLBk`), **M5 Newsletter Tracker** (`HnX3llf1rW6qmlPb`), **M5b Newsletter Metrics** (`4Jpo2RzBE2K6i2mE`), **M5c Website Metrics** (`3ltdhOLhIdfHJ2At`), **M6 Instagram Tracker** (`OjNFapHb4c0sNsCH`). Rien ne se publie/n'est marqué Done/Approved sans Karter.

## 11. IDs
Workflow `git92S6gDU1hGZ2b` · Editions DB `dad7f826891f4c2980e4183c59bac8b0` / ds `61418ed4-fb84-48fc-b372-247fbabe0c33` · Tasks DB `cd210f519f9a4d67a50ef6f50e851dc5` · cred Notion `AFTRSN-n8n-Backstage` `sCsj206WVkNxO0Jl` · Sentinel `xzaH0uWy0idKVphF` · Iris/Channel Content `MWUwLi3vu2Ws65lL`. Édition test permanente EP05 `39e24f2fef0c812ab3bbe8636418a482` (Done, gelée). Mode d'emploi wiki `3a024f2f-ef0c-819e-8a8c-d5dc126ba8a9`.

---

## Difficultés rencontrées
1. Trigger Notion non déclenchable par API et fragile en multi-items ; il ne surveille pas un champ précis mais toute la page.
2. Distinguer « annonce déjà faite » de « rien à faire », et savoir quand une édition est mûre pour l'annonce.
3. Rejouer indéfiniment sur des éditions passées/en cours.
4. Affamer les vagues au-delà de la première dans une boucle mal drainée.

## Solutions implémentées
1. **Refactor en scan planifié 1 min** : reprise de toutes les éditions éligibles à chaque tick ; filtrage « ne réagir qu'au line-up » par **empreinte** (`Line-up seen`) dans le flux.
2. **Garde `hasDateVenue`** : l'Announce ne part que quand date ET lieu sont connus ; décision de vague fondée sur la **checklist existante** (pas seulement l'empreinte).
3. **Gel** des éditions `Live`/`Cancelled`/`Done` (`Active editions`).
4. **Boucle `splitInBatches` batchSize 1** + reboucle en fin de run (drainage 1/tick).

## Lessons learned
1. **`scan planifié` > `Notion Trigger`** pour un moteur multi-items : testable, robuste, reprend tout l'éligible ; l'empreinte + un champ mémoire remplacent la « surveillance d'un champ ».
2. **Gate en jalons métier** (`hasDateVenue`, puis empreinte line-up), pas en simple présence/absence.
3. **Geler les états terminaux** évite de repolluer les éditions passées/live.
4. **Multi-items = parcours indexé + drainage 1/tick** ; `.first()` est un anti-pattern.
5. **Nommer juste** : watcher (workflow déterministe) ≠ agent (LLM). « Iris » = l'agent Channel Content, appelé par M2, pas le watcher.
6. **Toujours partir de la doc à jour** avant de toucher au système : l'état réel prime sur toute mémoire de session.
