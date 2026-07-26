---
type: raw
title: "POS-EXACT_Backstage-v3_N5-sidebar_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_n5-sidebar_2026-07-16.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · N5 « Nettoyage sidebar » (renommage des wrappers de linked views)

Date : 16.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **N5 FAIT & VALIDÉ par Karter le 16.07.2026**
Référence : `SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md` (§4 N5)

## Objectif

Renommer les 6 wrappers anonymes de linked views (« View of Tasks » ×4, « View of Editions », « View of Content & Social ») pour une sidebar lisible qui raconte la structure, + vérifier les icônes des pages.

## Marche à suivre exacte

1. **Pourquoi PAS l'API** : le wrapper d'une linked view est un objet « database » dont la seule data source est la base SOURCE partagée — `update-data-source` avec l'ID du wrapper renommerait la base **Tasks elle-même** (elle n'a qu'une data source, l'outil résout dessus). Trop risqué → navigateur.

2. **Navigateur (Claude in Chrome)** : ouvrir la sidebar, déplier The Backstage (clic sur le chevron de l'item — un clic sur le nom NAVIGUE au lieu de déplier).

3. **Pour chaque wrapper** : clic droit sur l'item sidebar → **Rename** → `cmd+a` → taper le nouveau nom → `Return` :
   - « View of Tasks » (1ᵉʳ, sous The Backstage) → `⏭️ Up Next`
   - « View of Editions » → `🎪 Upcoming Editions`
   - « View of Content & Social » → `📣 Publishing Pipeline`
   - « View of Tasks » (2ᵉ, sous The Backstage) → `✅ Awaiting validation`
   - Sous Agents Space : « View of Tasks » (1ᵉʳ) → `📋 Activity` · (2ᵉ) → `⏳ To validate`

4. **Désambiguïser les doublons « View of Tasks »** : le clic droit sur l'item sidebar SURLIGNE le bloc correspondant dans la page → on sait lequel on renomme. (Autre repère : l'ordre sidebar = l'ordre des blocs dans la page.)

5. **Vérif icônes/en-têtes** : Human Space 👩🏽‍💼 · Agents Space 🤖 · Operations ⚙️ · Knowledge 📚 · Archive 🗄️ — cohérents, rien à changer.

6. **Validation Karter** (16.07) → POS + coche + maj mémoire.

## Difficultés rencontrées

- Renommage des wrappers non couvert par l'API (risque de renommer la data source partagée à la place).
- Deux wrappers homonymes « View of Tasks » sous la même page — lequel est lequel ?
- Le clic sur un item sidebar navigue au lieu de déplier (il faut viser le chevron), et le popup « Invite members » masquait le bas de la sidebar.

## Solutions implémentées

- Renommage 100 % navigateur : clic droit → Rename → cmd+a → nouveau nom → Return (6 fois).
- Désambiguïsation par le surlignage du bloc lors du clic droit + ordre sidebar = ordre page.
- Fermer le popup (X) et scroller la sidebar pour atteindre tous les items.

## Lessons learned

- **Le NOM d'un wrapper de linked view est distinct du nom de la VUE et du nom de la DATA SOURCE** : la vue se renomme par API (`update-view name`), la data source aussi (`update-data-source title` — ⚠️ partagée !), mais le wrapper = navigateur uniquement.
- **Ne jamais passer l'ID d'un wrapper à update-data-source** : sur une base mono-source il résout vers la source partagée → renommage global involontaire.
- Une sidebar qui raconte la structure (emoji + vrai nom) vaut le quart d'heure de renommage : c'est elle que l'utilisateur voit en premier, avant même d'ouvrir les pages.
