---
type: raw
title: "POS-GENERIQUE_runner-tracker-livrable-humain-checklist-idempotente-source-unique_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-tracker-livrable-humain-checklist-idempotente-source-unique_2026-07-18.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Runner « tracker » d'un livrable produit par un humain : checklist idempotente + source unique par rollup

**Date :** 18.07.2026 · **Patron réutilisable** (sans IDs spécifiques). Dérivé de M3 (Visual Tracker AFTRSN), lui-même calqué sur le patron runner M2.

---

## 1. Quand utiliser ce patron
Quand un **livrable est produit par un humain** (ou un outil externe non automatisable par API), mais qu'on veut malgré tout :
- une **checklist automatique** « à faire / fait » alimentée par le moteur de tâches,
- un **catalogue durable** du livrable (statut + liens) relié à l'entité parente,
- **sans** que l'automate ne produise ni ne clôture le livrable.

C'est le cas typique quand une API tierce est **gated** (plan insuffisant) ou trop risquée : on ne génère pas, on **suit**.

## 2. Principe : source unique + rollup (« mis une fois, repris partout »)
Tout attribut qui appartient à l'**entité parente** (un lien de dossier, une référence, un budget) se pose **une seule fois sur l'entité**, et se **lit ailleurs par rollup** — jamais recopié ligne par ligne.
- Ajouter le champ sur l'entité parente (ex. URL `Dossier racine`).
- Dans la base « catalogue/checklist », ajouter un **rollup** via la relation → l'attribut de l'entité (`ROLLUP('<relation>', '<champ cible>', 'show_original')`).
- Résultat : chaque ligne affiche l'attribut de l'entité automatiquement.

## 3. Principe : checklist ≠ fiche de l'entité
Deux objets distincts, reliés mais jamais fusionnés :
- **La fiche de l'entité** = l'objet métier (l'événement, le projet…).
- **La checklist / fil conducteur** = la liste des livrables à produire, un par un, avec un état fait/pas-fait. Vit dans la base « tâches » et/ou une base « catalogue » avec une vue dédiée.

## 4. Schéma (additif, ne rien casser)
- Sur l'**entité parente** : le/les champ(s) « source unique » (ex. URL).
- Sur la base **catalogue** (ex. « assets ») : un axe **catégorie/segment** (ex. `Wave`, en **text** si les valeurs sont ouvertes/non bornées — évite le refus d'option select inconnue à l'écriture API) + un axe **format/type** (select borné) + un **Status** dont une valeur = clôture humaine + un **rollup** de l'attribut source unique + la **relation** vers l'entité.
- Une **vue checklist** : groupée par entité, filtrée par format, colonnes segment / status / rollup.

## 5. Workflow runner (nœuds)
1. **2 triggers** sur la base de tâches (Added + Updated), poll régulier, `simple:true`.
2. **Read the task** (get databasePage par `{{ $json.id }}`).
3. **Should track?** (code) : garde-fou d'éligibilité = `titre contient "<libellé du job>"` && `Status = "To Do"` && `Executor contient "Agent"` && `relation entité non vide`. Sinon `return []`. Extraire l'`entityId` (32 derniers hex de la relation) et le `segment` (ex. `titre.split(" · ")[0]`).
4. **Read the entity** (get) pour le nom / attributs.
5. **Build + filter** (code) : composer le titre du livrable + construire le **filtre de recherche** JSON (segment + format + relation contains entityId).
6. **Check existing** (HTTP POST vers l'API de requête de la base catalogue ; auth `predefinedCredentialType`, header de version ; corps = le filtre).
7. **Create if new?** (code) : si `results.length > 0` → `return []` (skip idempotent) ; sinon passer les métadonnées.
8. **Create row** (create databasePage) : Status = état initial « à produire », relation entité = `[entityId]`, segment, format, titre. **Jamais l'état de clôture** (réservé à l'humain).

Rattacher le **Error Workflow** (sentinelle) et l'`executionOrder v1`.

## 6. Idempotence : deux stratégies
- **Mutation d'état** (comme un runner qui produit un brouillon) : l'automate bascule le statut de la tâche, si bien que le filtre ne matche plus au repoll. Simple, mais **mute la tâche**.
- **Requête d'existence** (ce patron) : avant de créer, interroger la base catalogue (relation + segment + format). Skip si présent. **Ne mute pas la tâche** → à préférer quand le livrable est un travail humain et que la tâche doit rester le fil conducteur intact.

## 7. Test (déterministe) puis activation
- Construire le workflow **inactif** avec un **harnais Manual Trigger jetable** qui **injecte l'ID cible** (les triggers Manual/Notion ne se déclenchent pas par API).
- Faire cliquer « Test workflow » par le pilote ; **lire l'exécution par API**.
- Vérifier : (a) création correcte, (b) rollup rempli, (c) **2ᵉ run = skip** (idempotence, zéro doublon).
- Retirer le harnais (removeNode), **activer**, revalider (0 erreur, triggers présents).

## 8. Difficultés rencontrées (génériques)
1. **Hypothèse de plan/API fausse** : une API supposée disponible est en réalité gated (plan supérieur requis).
2. **Duplication d'un attribut** entre la ligne et l'entité → incohérence.
3. **Idempotence** quand on ne veut pas toucher l'objet déclencheur.
4. **Données de test qui masquent le comportement** (une ligne pré-existante fait « skip » le test de création).

## 9. Solutions implémentées (génériques)
1. **Repivoter de « générateur » vers « tracker »** dès que l'API de production est indisponible : suivre l'état plutôt que produire.
2. **Source unique + rollup** ; supprimer/ignorer les champs redondants par ligne.
3. **Garde-fou par requête d'existence** avant création ; aucune mutation du déclencheur.
4. **Décaler la donnée de test** sur un segment vierge pour prouver la création, garder une donnée existante pour prouver l'idempotence.

## 10. Lessons learned (génériques)
- **Vérifier les prérequis d'une API tierce avant de coder** (plan, quotas, scopes).
- **« Mis une fois, repris partout »** : attribut d'entité → sur l'entité, lu par rollup.
- **Séparer checklist (fil conducteur) et fiche de l'entité** : deux objets, reliés, jamais fusionnés.
- **Choisir consciemment le point d'ancrage de l'idempotence** (mutation vs requête) selon qui produit le livrable (machine vs humain).
- **Un livrable humain reste humain** : l'automate prépare et suit, il ne produit ni ne clôture (invariant « jamais l'état final par l'automate »).
