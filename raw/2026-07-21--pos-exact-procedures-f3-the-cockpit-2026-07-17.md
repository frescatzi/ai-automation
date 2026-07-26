---
type: raw
title: "POS-EXACT_Procedures_F3-the-cockpit_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_procedures_f3-the-cockpit_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Fiche procédure « The Cockpit » (wiki 📚 Knowledge, AFTRSN Backstage)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Type : POS-EXACT (marche à suivre réelle, avec IDs et appels API).
POS-GÉNÉRIQUE : partagé pour tout le lot, voir `POS-GENERIQUE_creer-fiche-procedure-wiki-notion-mvp_2026-07-17.md` (même méthode de build).

## Contexte
Lot 1 « trio humain », fiche 3 sur 3. How-to « The Cockpit » : comment LIRE le tableau de bord de Team Space. Ferme la boucle du trio (créer une tâche, valider un agent, lire le cockpit).

## Cible et IDs
- Fiche créée : `3a024f2f-ef0c-81dd-84ec-f85cf830f6d4`, catégorie How-to.
- Références (par lien, jamais copiées) : Team Space `39124f2fef0c811599edec1c18b44b20` (section 🔭 Cockpit) et ses blocs : Up Next `39e24f2fef0c817ab3b5f50fee65eb1f`, Upcoming Editions `39e24f2fef0c81d9b9bde18dff28b75c`, Publishing Pipeline `39e24f2fef0c81519288e8db47d61f53`, Awaiting validation `39f24f2fef0c81209f84e79ca8c809b3`. Index 📚 Knowledge `39f24f2fef0c814eae67e7a829f3c8c2`.

## Étapes réalisées
1. `notion-create-pages`, parent = ds Knowledge. Format MVP : When to use / Where it is / What each part shows / How to read it daily / Common mistakes / Owner / Related. Vocabulaire à jour, zéro tiret cadratin.
2. Convention « Related = titres cliquables » BOUCLÉE sur le trio :
   - Fiche 3 → Fiche 1 et Fiche 2 : liens directs dès la création.
   - Mise à jour des fiches 1 et 2 (`update_content`) : « The Cockpit » (texte) remplacé par un lien direct vers `3a024f2f-ef0c-81dd-84ec-f85cf830f6d4`.
   - Résultat : chaque fiche du trio pointe vers les deux autres en liens directs cliquables.
3. Validation Karter : ✅ 17.07.2026.

## Points de contenu (le pourquoi)
- Cockpit cadré comme un ÉCRAN DE LECTURE (voir ce qui presse / en retard / à venir / à valider), pas un lieu de construction. Chaque section renvoie à la SOP d'action correspondante (Task Lifecycle pour créer/déplacer ; Validate an Agent's Work pour vider la file).
- Toutes les vues du cockpit = fenêtres filtrées sur les mêmes données (Tasks/Editions), jamais des copies.

## Difficultés rencontrées
- Aucune bloquante.

## Solutions implémentées
- Réutilisation des IDs de blocs cockpit déjà connus (mémoire/spec) pour décrire fidèlement chaque vue.
- Câblage Related bouclé en fin de trio par petits `update_content` ciblés.

## Lessons learned
- Documenter le trio dans l'ordre créer → valider → lire donne une boucle pédagogique cohérente : chaque fiche renvoie aux deux autres.
- Le câblage Related bidirectionnel se termine naturellement à la dernière fiche (toutes les cibles existent alors).
