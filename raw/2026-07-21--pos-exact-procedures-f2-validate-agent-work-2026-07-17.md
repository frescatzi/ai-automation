---
type: raw
title: "POS-EXACT_Procedures_F2-validate-agent-work_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_procedures_f2-validate-agent-work_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Fiche procédure « Validate an Agent's Work » (wiki 📚 Knowledge, AFTRSN Backstage)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Type : POS-EXACT (marche à suivre réelle, avec IDs et appels API).
POS-GÉNÉRIQUE : partagé pour tout le lot, voir `POS-GENERIQUE_creer-fiche-procedure-wiki-notion-mvp_2026-07-17.md` (même méthode de build ; on ne duplique pas, un seul générique pour les trois fiches).

## Contexte
Lot 1 « trio humain », fiche 2 sur 3. SOP « Validate an Agent's Work » : le geste humain de relecture/validation du travail d'un agent, côté humain du pont des deux mondes.

## Cible et IDs
- Fiche créée : `3a024f2f-ef0c-8188-9ba7-e120a5127e5e`, catégorie SOP.
- Références (par lien, jamais copiées) : Team Space `39124f2fef0c811599edec1c18b44b20` (section ✅ Awaiting validation, bloc To Validate `39f24f2fef0c81209f84e79ca8c809b3`) ; Agents Space `39124f2fef0c81a99ba6e84df86498ff` (vue ⏳ To validate) ; index 📚 Knowledge `39f24f2fef0c814eae67e7a829f3c8c2`.

## Étapes réalisées
1. `notion-create-pages`, parent = ds Knowledge `7f499c43-...`. Format MVP : When to use / Where to find work to validate / Validate a task (5 étapes) / The rule that makes this work / Common mistakes / Owner / Related. Vocabulaire à jour dès la création (Team Space, 🤖 Agent, Team Member) ; zéro tiret cadratin.
2. Convention « Related = titres cliquables » appliquée dans les DEUX sens :
   - Fiche 2 → Fiche 1 : lien direct « 🔄 Task Lifecycle » (`3a024f2f-ef0c-81e6-8d24-db5a79a8630c`).
   - Fiche 1 → Fiche 2 : `notion-update-page` (update_content) sur la fiche 1, « Validate an agent's work » (texte) remplacé par un lien direct vers `3a024f2f-ef0c-8188-9ba7-e120a5127e5e`.
   - « The Cockpit » reste en texte dans les deux fiches, à convertir en lien direct dès la fiche 3 créée.
3. Validation Karter : ✅ 17.07.2026.

## Points de contenu (le pourquoi)
- « Same tasks, no copies » : la file du Cockpit (✅ Awaiting validation) et la vue ⏳ To validate d'Agents Space montrent les MÊMES tâches via des vues filtrées. Traduction quotidienne de l'invariant zéro-duplication du build.
- Règle du pont : un 🤖 Agent s'arrête toujours à To Validate ; seul un Team Member met Done. Prépare proprement la délégation n8n (un agent n'écrit jamais Done).

## Difficultés rencontrées
- Aucune bloquante. Seule vigilance : rester cohérent avec le vocabulaire Team Space / 🧑 Team Member issu du renommage live effectué dans une autre session.

## Solutions implémentées
- Partir directement du bon vocabulaire (mémoire projet à jour) au lieu de créer puis corriger.
- Câbler le lien Related dès la création, la fiche cible (fiche 1) existant déjà.

## Lessons learned
- Une fois la convention Related posée, le câblage bidirectionnel se fait dès qu'une fiche cible existe (ID stable, robuste aux futures éditions).
- Le POS-GÉNÉRIQUE de build est partagé entre fiches (même méthode) : on ne produit qu'un POS-EXACT par fiche, pas un générique redondant à chaque fois.
