---
type: raw
title: "POS-GENERIQUE_runner-tracker-consommer-tache-checklist-remplir-livrable-pret-a-coller-gate-humain_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-tracker-consommer-tache-checklist-remplir-livrable-pret-a-coller-gate-humain_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-GÉNÉRIQUE · Runner-tracker : consommer une tâche de checklist et la remplir d'un livrable prêt à coller, quand la plateforme cible n'a pas d'API de création

**Date :** 19.07.2026 · **Contexte d'origine :** AFTRSN P3 · M5 Newsletter Tracker · **Type :** POS-GÉNÉRIQUE (réutilisable).

> Patron : une plateforme de diffusion (emailing, design, réseau social…) **n'expose pas d'API pour créer/éditer le livrable** → l'automate ne fabrique pas le livrable, il **prépare tout ce qu'il faut** et **suit** l'avancement via une tâche de checklist, l'humain exécutant le geste natif derrière son gate.

---

## 1. Difficultés (ce qui bloque)
1. **Pas d'API de création côté cible.** Vérifier tôt les *méthodes réellement exposées* (ex. Wix Email Marketing : seulement Get/List/Publish/SendTest/Reuse — pas de Create/Update). Une hypothèse « il y a sûrement un create » coûte cher si elle est fausse. → Trancher la faisabilité AVANT de choisir l'architecture.
2. **Relations Notion peu fiables en lecture.** Un champ relation peut revenir **vide** (`relation:[]`) à la fois via le nœud n8n Notion `getAll` ET via l'API legacy `POST /v1/databases/{id}/query` (constaté quand la page cible est archivée). Seule l'API data-source récente la voit. → Ne pas fonder le matching sur une relation lue en scan.
3. **Multi-items / drainage** (voir POS `M3-M4-fiabilisation-scan`).

## 2. Solutions (le patron)
1. **Faisabilité d'abord.** Interroger le spec de l'API cible (méthodes exposées). Si pas de create/update → basculer en **tracker** (l'humain compose, l'automate prépare+suit). Même logique que M3 (Canva).
2. **Consommer une tâche existante, ne pas en créer.** Si un orchestrateur amont (ici M1 Edition Watcher) pose déjà une checklist par vague avec une case dédiée (« Newsletter », « Wix event page », « Flyer/visual »…), le runner **consomme** cette tâche : il la remplit et la fait avancer `To Do → To Validate`. Filtrer les tâches par `Status=To Do` + titre contenant le nom du job.
3. **Matcher par TITRE, pas par relation.** Quand la tâche ET la source (copie) portent tous deux, dans leur **titre**, le nom d'entité + la sous-clé (édition + vague), matcher là-dessus :
   - tâche `<vague> · <job> · <édition>` → `vague = split(' · ')[0]`, `édition = slice(2).join(' · ')`.
   - source `<préfixe> · <édition> · <vague>` → `vague = dernier segment`, `édition = slice(1,-1).join(' · ')`.
   Robuste, indépendant des relations. (Prévoir les doublons de source → prendre la première.)
4. **Requête unique côté source, filtrée par le gate.** Récupérer les sources en **une** requête HTTP filtrée sur le statut de gate (ex. `Status=Approved`) plutôt qu'une requête par tâche. Le filtre de la requête EST le gate.
5. **Livrable prêt-à-coller dans un champ texte.** Assembler les morceaux (sujet/preview/corps…) en un bloc texte lisible dans le champ `Notes` de la tâche (rich_text ; corps courts < 2000 car → un seul objet). L'humain copie-colle en un geste.
6. **Drainage = mutation de la tâche.** Mettre à jour la tâche (`Status=To Validate` + `Notes`) = le drainage ; le filtre `To Do` empêche la reprise. Multi-items pour tout drainer en un tick. Source absente/non-gatée ⇒ **skip** (la tâche reste To Do, réessai).
7. **Invariants** : l'automate n'écrit jamais `Done`/`Approved` ni n'envoie ; rien ne sort sans l'humain ; secrets = propriétaire ; Error-WF = Sentinel ; workflow créé **inactif**, activé après validation.

## 3. Lessons (à retenir)
- **Vérifier le spec de l'API cible avant de promettre une automatisation de création.** « Diffuser » ≠ « composer » : beaucoup d'API (emailing, design) gèrent l'envoi/gestion d'un objet existant mais pas sa création de contenu.
- **Ne jamais dépendre d'une relation Notion lue en scan.** Matcher par des champs *portés par l'objet lui-même* (titre, texte). Auditer les runners existants qui lisent des relations (ici : M3 `property_event`, M4 relation Edition de la copie) — risque de famine silencieuse si la relation revient vide.
- **Le filtre de la requête source doit encoder le gate** (ex. `Approved`) : un seul endroit à changer si le gate évolue.
- **Un tracker apporte de la valeur même sans rien créer** : il évite à l'humain de chercher la bonne source, la lui sert prête, et rend l'étape visible/traçable derrière le gate.
- **Tester les DEUX faces du gate en live** (source gatée → rempli ; non-gatée/absente → reste To Do) sur des données de test isolées, puis nettoyer (archiver la source de test via API `PATCH archived:true` — l'outil MCP Notion ne sait pas archiver).
