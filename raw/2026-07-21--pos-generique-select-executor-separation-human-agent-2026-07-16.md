---
type: raw
title: "POS-GENERIQUE_select-executor-separation-human-agent_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_select-executor-separation-human-agent_2026-07-16.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Séparer travail humain / travail agent dans une base de tâches Notion via un select « Executor »

Date : 16.07.2026 · Portée : n'importe quel workspace Notion (toute marque/équipe) piloté par API (MCP Notion) 
Principe : **une seule base de tâches, zéro duplication** — la séparation des mondes est portée par une simple propriété select, et les espaces dédiés ne sont que des vues filtrées.

## Contexte d'usage

Vous voulez qu'humains et agents IA travaillent dans la même base de tâches, avec des espaces distincts (dashboard humain / dashboard agents), sans copier de données. Le contrat : chaque tâche déclare *qui l'exécute*.

## Marche à suivre

1. **Lire le schéma avant de modifier** : fetch de la data source (`collection://<ds_id>`) pour confirmer que la propriété n'existe pas déjà et que vous êtes dans le BON espace (vérifier l'ancestor-path — attention aux pages homonymes entre teamspaces).

2. **Créer le select** via update-data-source :
   ```sql
   ADD COLUMN "Executor" SELECT('🧑 Human':blue, '🤖 Agent':purple)
   ```
   - Colonne neuve → pas besoin de relister quoi que ce soit.
   - ⚠️ Si un jour vous MODIFIEZ ce select (`ALTER COLUMN … SET SELECT`), relistez TOUTES les options existantes (mêmes noms → IDs préservés), sinon elles sont supprimées.
   - Deux couleurs contrastées = l'intervention agent se repère d'un coup d'œil dans les vues humaines.

3. **Backfill des tâches existantes** : query SQL pour lister les pages (`SELECT url, "Task" FROM "collection://<ds_id>"`), puis update-page `update_properties` `{"Executor": "🧑 Human"}` sur chacune. Re-query de contrôle après coup.

4. **Rendre la pastille visible dans les vues clés** : update-view avec `SHOW` en **relistant toutes** les propriétés affichées (SHOW remplace la liste entière ; l'ordre passé = l'ordre affiché). Placer Executor juste après le titre pour qu'elle apparaisse en tête de carte/ligne. Les vues « tout afficher » intègrent automatiquement la nouvelle propriété.

5. **Valeur par défaut via le template** : les selects Notion n'ont pas de défaut au niveau schéma — le défaut se porte sur le **template de nouvelle page**. Un template est une page ordinaire de la collection : update-page `update_properties` `{"Executor": "🧑 Human"}` sur l'ID du template suffit (pas besoin du navigateur). Vérifier par re-fetch du template.

6. **Tester en conditions réelles** : créer une tâche via le bouton « New » (qui applique le template par défaut) → elle doit naître avec Executor pré-rempli. Supprimer la tâche test (corbeille = réversible).

7. **La suite logique** (hors périmètre de ce POS) : l'espace agents = linked views filtrées `Executor = Agent` ; le pont humain↔agent = un statut de validation visible des deux côtés.

## Difficultés rencontrées

- Croyance héritée : « les templates de base ne sont pas pilotables par API » — vrai pour les créer/réordonner, faux pour éditer leurs propriétés.
- `SHOW` remplace la liste complète des propriétés affichées : risque de faire disparaître des colonnes si on ne reliste pas tout.

## Solutions implémentées

- Édition du template par `update_properties` directement sur sa page (un template EST une page de la collection) — vérifiée par re-fetch.
- Relisting systématique de toutes les displayProperties à chaque `SHOW`, nouvelle propriété insérée à la position voulue.

## Lessons learned

- **Le défaut d'un select vit dans le template, pas dans le schéma** — c'est le template qu'il faut éditer, et c'est faisable par API.
- **Une propriété + des vues filtrées > des bases séparées** : la séparation des mondes ne justifie jamais une copie de données ; une seule vérité, plusieurs fenêtres.
- **Toujours re-vérifier après écriture** (re-query / re-fetch) : le coût est marginal et ça attrape les échecs silencieux.
- **SHOW est déclaratif et exhaustif** : penser « voici la liste finale ordonnée », pas « j'ajoute une colonne ».
