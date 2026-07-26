---
type: raw
title: "POS-GENERIQUE_deplacer-blocs-notion-navigateur-move-to_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_deplacer-blocs-notion-navigateur-move-to_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: NOTION
immutable: true
---

# POS-GÉNÉRIQUE — Notion : restructurer des pages SANS API (fallback navigateur « Move to »)

Date : 17.07.2026 · Portée : générique, applicable à tout espace Notion (aucun ID spécifique)

## Quand l'utiliser

Quand le connecteur/l'API Notion est indisponible (jeton expiré, réautorisation non propagée à la session en cours, MCP déconnecté) et qu'il faut quand même déplacer des blocs entre pages, dissoudre une sous-page, ou réorganiser une structure. Prérequis : un navigateur piloté (Claude in Chrome) dont la session Notion est déjà authentifiée.

## La procédure « Move to » (équivalent navigateur de move-pages)

1. Ouvrir la page SOURCE.
2. Survoler le bloc à déplacer → cliquer la poignée **⋮⋮** qui apparaît à gauche.
3. Dans le menu : **Move to** → taper le nom de la page destination → **vérifier le sous-titre du résultat** (chemin teamspace/parent) avant de cliquer — les homonymes entre teamspaces sont le piège n°1.
4. Répéter bloc par bloc. **Règle d'or : chaque bloc arrive EN FIN de page destination → l'ordre des moves = l'ordre final.** Planifier la séquence avant de commencer.
5. Marche pour : bases (databases), linked views, headings, paragraphes — tout type de bloc.

## Compléments utiles

- **Corbeiller une page devenue vide** : ••• (haut droite) → Move to Trash. Les liens `<page>` qui pointaient vers elle disparaissent automatiquement des pages parentes. Récupérable ~30 jours.
- **Insérer un bloc AU-DESSUS d'un bloc existant** (pour ajouter un titre de section au-dessus d'un bloc déplacé) : survoler le bloc → **alt+clic sur « + »** → Escape (le slash-menu s'ouvre tout seul, le fermer) → taper. `## ` + texte = H2 ; `cmd+a` puis `cmd+i` = ligne en italique.
- **Toast de confirmation** : chaque move affiche « Moved <bloc> to <page> » avec Undo — vérifier ce toast avant d'enchaîner.

## Pièges connus du pilotage navigateur sur Notion

- **Premier caractère avalé** dans les champs de recherche : vérifier le texte tapé, `cmd+a` + retaper si tronqué.
- **Scroll souvent bloqué** (« page still loading » côté automation ; les boards à scroll horizontal capturent la molette) : les clics/hover/frappe/captures passent toujours → `cmd+Down` pour descendre, replier des sections pour tout voir, zoom de région pour lire les petits éléments.
- **Cliquer un item de sidebar navigue** au lieu de déplier → viser le chevron.
- Homonymes multi-teamspaces dans TOUS les pickers (Move to, recherche) : toujours lire le chemin.

## Difficultés rencontrées

- Réautorisation d'un connecteur OAuth non propagée à la session en cours : les outils API restent morts alors que l'interface web dit « Connecté ».
- Séquençage : un bloc déplacé au mauvais moment arrive au mauvais endroit (fin de page) et doit être re-déplacé ou re-ordonné à la main.

## Solutions implémentées

- Bascule immédiate au navigateur après 2-3 tentatives de reconnexion espacées — ne pas bloquer l'utilisateur sur un problème d'infrastructure.
- Ordre des moves calculé À L'AVANCE pour reproduire exactement la structure cible sans drag & drop (le drag est le geste le moins fiable en automation).

## Lessons learned

- **Le navigateur est un fallback COMPLET pour la restructuration** : moves, suppressions, titres, formatage — tout est faisable sans API, juste plus lentement (compter ~30 s par bloc).
- L'ordre-d'arrivée-en-fin-de-page transforme une réorganisation complexe en simple liste ordonnée de moves — aucun repositionnement nécessaire si la séquence est bonne.
- Toujours annoncer à l'utilisateur le mode dégradé et pourquoi (confiance + il peut réparer le connecteur pour la prochaine session).
