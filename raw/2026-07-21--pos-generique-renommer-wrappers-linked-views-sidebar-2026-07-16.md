---
type: raw
title: "POS-GENERIQUE_renommer-wrappers-linked-views-sidebar_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_renommer-wrappers-linked-views-sidebar_2026-07-16.md"
captured: 2026-07-21
vault: brands
brand: NOTION
immutable: true
---

# POS-GÉNÉRIQUE — Renommer les wrappers « View of… » des linked views Notion (sidebar propre)

Date : 16.07.2026 · Portée : tout workspace Notion avec des linked views créées par API ou via « /linked »

## Contexte d'usage

Chaque linked view créée engendre un wrapper nommé par défaut « View of <Base> » qui pollue la sidebar et les recherches. Vous voulez que chaque item de la sidebar porte le nom réel de la vue (avec emoji), pour que l'arborescence raconte la structure.

## Marche à suivre

1. **Ne PAS tenter l'API** : le wrapper est un objet « database » mono-source dont la source est la base PARTAGÉE. Un update-data-source avec l'ID du wrapper renommerait la base source pour tout le monde. Le renommage du wrapper est navigateur uniquement. (À distinguer : le nom de la VUE se change par API `update-view name` ; ici on parle de l'étiquette sidebar.)

2. **Navigateur** : déplier la page parente dans la sidebar en cliquant le **chevron** (cliquer le nom navigue au lieu de déplier). Fermer les popups qui masquent le bas de la sidebar.

3. **Pour chaque wrapper** : clic droit → **Rename** → tout sélectionner (`cmd+a`) → taper `emoji + nom de la vue` → `Return`.

4. **Doublons homonymes** (« View of Tasks » ×n sous la même page) : le clic droit **surligne le bloc correspondant dans la page** — c'est le désambiguïsateur fiable. Repère secondaire : l'ordre sidebar = l'ordre des blocs dans la page.

5. **Passe finale** : vérifier les icônes des pages de l'arborescence (cohérence emoji/rôle), puis relire la sidebar comme un sommaire : elle doit raconter la structure sans ouvrir une seule page.

## Difficultés rencontrées

- Renommage wrapper absent de l'API ; tentation dangereuse de passer l'ID du wrapper à update-data-source.
- Wrappers homonymes indiscernables par le nom seul.
- Clic sidebar qui navigue au lieu de déplier ; popups masquant les items.

## Solutions implémentées

- Clic droit → Rename systématique (4 gestes par wrapper).
- Désambiguïsation par surlignage du bloc + ordre.
- Viser le chevron pour déplier ; fermer les popups d'abord.

## Lessons learned

- **Trois noms distincts autour d'une linked view** : la data source (partagée — API, danger), la vue (API, sans risque), le wrapper sidebar (navigateur only). Savoir lequel on touche évite un renommage global accidentel.
- La sidebar est la première interface : le nettoyage des wrappers est un vrai livrable, pas du cosmétique gratuit.
