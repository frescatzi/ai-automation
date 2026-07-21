---
type: raw
title: "POS-GENERIQUE_verifier-pont-humain-agent-taches-sonde_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_verifier-pont-humain-agent-taches-sonde_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Vérifier bout-en-bout un pont humain/agent par tâches sonde

Date : 17.07.2026 · Portée : générique — applicable à tout espace Notion (ou autre outil à vues filtrées) où une base unique est partagée entre un « monde humain » et un « monde agent » via une propriété de séparation (ex. select `Executor`) et des vues filtrées. Aucune référence à des IDs spécifiques.

---

## Principe

Un « pont » = une base unique + des vues filtrées de part et d'autre (zéro copie de données). Pour prouver qu'il fonctionne, on injecte des **tâches sonde** — une par cas de partition — puis on vérifie mécaniquement leur présence/absence dans chaque vue. La vérification devient une matrice binaire, rapide et sans ambiguïté.

## Marche à suivre

1. **Re-fetch de l'état réel avant tout** : récupérer le schéma de la base (propriété de séparation et ses options, statuts, relations, rollups). Ne jamais se fier à la doc seule — l'outil fait foi.
2. **Définir les cas de partition** : lister les combinaisons discriminantes des filtres (ex. humain+relation, humain+sans relation, agent+statut de validation). Une sonde par cas, pas plus.
3. **Créer les sondes par API** avec un préfixe évident (`TEST <milestone> - …`) et une note indiquant leur sort prévu (« à corbeiller après validation » / « conservée comme pilote »).
4. **Interroger chaque vue filtrée par API** (exécution de la vue avec ses filtres réels, pas une requête reconstruite) et remplir la matrice attendu/observé :
   - chaque sonde apparaît dans les vues où elle DOIT être ;
   - chaque sonde est absente des vues où elle NE DOIT PAS être (aussi important que la présence) ;
   - le cas « pont » (agent + à valider) apparaît DES DEUX CÔTÉS.
5. **Vérifier l'intégrité relationnelle** : fetch d'une entité liée (ex. l'événement rattaché) — la relation inverse liste les sondes, les rollups/formules résolvent. Compléter par un contrôle visuel (les rollups rendus à l'écran).
6. **Nettoyer** : corbeiller les sondes devenues inutiles (via l'UI si l'API de suppression est risquée/indisponible), et **conserver délibérément** la sonde qui servira de pilote à la phase suivante s'il y en a une. Contre-vérifier par requête que la base est revenue à l'état attendu.
7. **Documenter et clôturer** : cocher le milestone, mettre à jour la mémoire/spec, produire le POS exact + ce générique.

## Garde-fous

- Les sondes ne doivent jamais modifier le modèle (pas de nouvelle option, pas de nouveau statut) : elles n'utilisent que l'existant.
- Vérifier l'ABSENCE autant que la présence : un filtre trop large se voit uniquement par les intrus.
- Une vue « file de validation » côté humain peut légitimement inclure les items humains ET agents — vérifier l'intention (file du décideur) avant de crier au bug.
- Suppression = geste réversible d'abord (corbeille), jamais de suppression définitive pendant la vérification.

---

## Difficultés rencontrées

1. Menus contextuels d'UI web = toggles : un clic prématuré (page en chargement) ne fait rien, un double clic referme le menu — automatisation navigateur fragile sans repère d'état.
2. Les connecteurs/outils d'accès peuvent avoir des périmètres différents de ce qu'on croit (un outil voit une partie du système, pas tout) — découvert en vérifiant les outils en début de session.

## Solutions implémentées

1. Séquence robuste : attendre un signal de chargement (titre de page correct), UN clic, localiser l'entrée de menu par l'arbre d'accessibilité (pas par coordonnées), cliquer la référence, puis contre-vérifier l'effet par requête API.
2. Vérification des connecteurs et de leur périmètre EN DÉBUT de session, et consignation de tout écart dans la doc de la phase suivante.

## Lessons learned

1. **Une sonde par cas de partition + une matrice présence/absence** = la vérification bout-en-bout la plus économe : quelques créations, quelques requêtes, un verdict binaire.
2. **Penser au recyclage des sondes** : celle qui correspond au cas de la phase suivante peut être conservée comme pilote — un objet déjà validé vaut mieux qu'un objet neuf.
3. API et visuel se complètent : l'API prouve les filtres et les données, l'écran prouve le rendu (rollups, pastilles, groupements). Valider les deux évite les faux « tout est vert ».
