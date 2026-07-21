---
type: raw
title: "POS-GENERIQUE_orchestrateur-n8n-entite-notion-vers-checklist-taches-liees-idempotente_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_orchestrateur-n8n-entite-notion-vers-checklist-taches-liees-idempotente_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Construire un orchestrateur n8n : entité Notion → checklist de tâches liées, en phases et idempotente

**Date :** 17.07.2026 · **Contexte d'origine :** M1 Phase 3 AFTRSN (workflow `AFTRSN-IRIS-EVENT-AGENT`, `git92S6gDU1hGZ2b`). Procédure réutilisable pour tout automate qui lit une entité Notion (édition, projet, client…) et génère une liste de tâches rattachées, sans jamais clôturer ni dupliquer.

## Quand l'utiliser
Dès qu'une source de vérité Notion (une ligne de base) doit engendrer automatiquement une checklist de tâches dans une autre base reliée, avec : un garde-fou de périmètre, une génération par « phases » selon l'état de la source, une relance sans doublon, et l'invariant « un automate n'écrit jamais Done ».

## Principes
1. **Notion = source + destination.** On LIT l'entité (jamais on n'y écrit son statut), on ÉCRIT les tâches dans la base Tasks reliée.
2. **Garde-fou de périmètre.** Tant que ce n'est pas validé, ne traiter qu'une entité désignée (ID en dur, vérifié dans le code) ou un filtre strict.
3. **Phases de contenu, pas blocages.** Si une info clé peut arriver plus tard, en faire une PHASE (ex. « avant / après » cette info) plutôt qu'un verrou : chaque phase produit son lot de tâches. Un champ de contrôle Notion (select, valeur vide = Auto) laisse l'humain forcer une phase ou laisser l'automate décider.
4. **Idempotence obligatoire.** Lire la checklist existante AVANT de créer, dédoublonner par nom exact, n'ajouter que le manquant. Une relance ne doit jamais dupliquer.
5. **Verrou métier si besoin.** Certaines transitions n'ont pas de retour arrière (ex. une fois l'info posée, on ne rejoue pas la phase « pré-info ») : le coder ET l'exposer côté Notion.
6. **Jamais Done.** Les tâches créées naissent en `To Do`, exécuteur = l'agent ; la clôture reste humaine.

## Étapes

1. **Repérer les IDs** (par API) : base source (data source), base Tasks (DB id pour le node Notion + ds id pour les requêtes), credential Notion dédié, workflow Sentinel (Error Workflow).
2. **Réutiliser un node Notion prouvé** : copier la config d'un workflow existant (resource `databasePage`, `typeVersion` courante, référence credential). Ne pas réinventer.
3. **Node lecture de l'entité** : `databasePage:get`, `pageId` en mode URL. Garde-fou : dans le code aval, comparer l'ID lu (sans tirets) à l'ID attendu, sinon `throw`.
4. **Node lecture de la checklist existante** : `databasePage:getAll`, `returnAll:true`, sur la base Tasks. Sert au dédoublonnage.
5. **Node Code « brief »** : référencer les nodes par NOM (`$('Read ...').first().json`), lire les propriétés en clés `property_snake_case` (+ `name` pour le titre), détecter l'état (info présente ou non), lire le champ de contrôle, calculer la ou les phase(s) à produire et un éventuel `blockedReason`.
6. **Node Code « plan »** : catalogue {phase → liste de libellés de tâches}, dédoublonnage contre la checklist existante (Set de noms), émettre uniquement les tâches nouvelles des phases ciblées. Sortie vide = 0 création (relance propre).
7. **Node Notion création** : `databasePage:create`, `title` en expression, propriétés en `Nom|type` ; **relation en tableau** (`={{ [$json.id] }}`), select en `selectValue`. Status = To Do, Executor = agent.
8. **Réglages** : Error Workflow = Sentinel, `executionOrder v1`. **Valider par API** (`n8n_validate_workflow`).
9. **Parité Notion** : si une règle métier vit dans le workflow, ajouter une **formule** sur la base source qui l'affiche (Notion ne bloque pas un select selon un autre champ ; la formule = parité visible).

## Boucle de test (trigger manuel)
Manual Trigger = non déclenchable par API. Donc : créer avec le node d'écriture **désactivé**, faire cliquer « Execute », lire l'exécution par API pour inspecter lecture + logique sans polluer, corriger, **activer** l'écriture, **recharger l'éditeur** (obligatoire), relancer, vérifier la création par requête SQL Notion.

## Pièges à éviter
- Exécuter sans recharger l'éditeur après une modif API → on rejoue l'ancien canvas.
- Lire les propriétés Notion par leur nom d'affichage → vide ; ce sont des clés `property_*`.
- Relation Notion écrite en chaîne → `value.relationValue.filter is not a function` ; passer un tableau.
- Champ Notion vide masqué sur la fiche → « Show empty properties ».
- Croire qu'une relance est sûre sans idempotence → doublons.
- Utiliser « — » dans les libellés générés (règle de style) → préférer « · ».

## Résultat attendu
Un orchestrateur relançable qui, d'un clic, transforme une entité en checklist rattachée, respecte les phases et le périmètre, ne duplique jamais, ne clôture jamais, et dont la logique est lisible côté Notion.
