---
type: raw
title: "POS-GENERIQUE_auto-declencher-orchestrateur-n8n-sur-changement-notion-empreinte-idempotente_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_auto-declencher-orchestrateur-n8n-sur-changement-notion-empreinte-idempotente_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Rendre un orchestrateur n8n autonome sur changement Notion, en ne réagissant qu'à un champ, via une empreinte idempotente

**Date :** 17.07.2026 · **Contexte d'origine :** Iris (auto-déclenchement de M1 AFTRSN). Réutilisable pour tout automate qui doit se déclencher sur les changements d'une base Notion mais **ne réagir qu'à un champ précis**, sans boucler ni dupliquer.

## Le problème
Un Notion Trigger voit **toute la page** à chaque changement : impossible de dire « déclenche-toi seulement si le champ X change ». Et si l'automate écrit lui-même dans la page, il risque de se re-déclencher en boucle.

## La solution en 3 briques
1. **Empreinte du champ surveillé.** Calculer une signature normalisée du champ X (texte : split/trim/lowercase/tri ; relation : IDs sans tirets, triés ; joindre par `|`). Vide si X est vide.
2. **État mémorisé.** Un champ texte caché sur la base (`<agent> <champ> seen`) stocke la dernière empreinte traitée. Comparaison à chaque réveil : empreinte courante == mémorisée → ne rien faire.
3. **Écriture bornée.** N'écrire la nouvelle empreinte QUE lorsqu'on a réellement agi (chaîner l'écriture après l'action, réduite à 1 item). Ainsi le re-déclenchement provoqué par sa propre écriture voit « empreinte identique » → 0 action, 0 écriture → la boucle s'arrête en un tour.

## Étapes
1. **Triggers** : ajouter les Notion Triggers utiles (`pageAddedToDatabase` et/ou `pagedUpdatedInDatabase`) sur la DB, credential dédié, `simple:true`, poll adapté. Les faire converger vers un node de lecture.
2. **Lecture** : `databasePage:get` avec `pageId = =https://www.notion.so/{{ $json.id }}` (id de la page déclenchante).
3. **Empreinte + décision** (Code) : calculer l'empreinte de X ; lire l'état mémorisé (`property_<champ>_seen`, avec fallback scan de clé) ; décider l'action. Fonder la décision sur l'**état déjà présent** (ex. tâches/enregistrements existants liés), pas seulement sur l'empreinte, pour survivre aux relances.
4. **Action** : produire les enregistrements (idempotents : dédoublonner par clé).
5. **Fin** : `Limit maxItems 1` après l'action → `databasePage:update` qui écrit la nouvelle empreinte dans le champ mémoire.
6. Error Workflow = Sentinel. **Valider par API**, tester par « Fetch Test Event » (trigger non déclenchable par API), garder **inactif** jusqu'à validation, puis activer.

## Pièges
- Ajouter un champ mémoire caché avant de coder (sinon `property_<...>` absent).
- Écrire l'empreinte à chaque réveil (même sans action) → boucle non bornée : n'écrire qu'après action.
- Se fier à l'empreinte seule pour distinguer « rien fait encore » de « rien changé » : croiser avec l'état existant.
- Notion Trigger = toute la page ; ne jamais supposer qu'il filtre un champ.
- Poll interval : réglable dans l'éditeur ; défaut souvent court.

## Résultat attendu
Un automate qui se réveille sur tout changement de la base mais n'agit qu'au vrai changement du champ surveillé, ne duplique pas, ne clôture pas, et ne boucle pas sur sa propre écriture (un tour à vide au maximum).
