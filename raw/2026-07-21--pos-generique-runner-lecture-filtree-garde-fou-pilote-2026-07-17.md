---
type: raw
title: "POS-GENERIQUE_runner-lecture-filtree-garde-fou-pilote_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-lecture-filtree-garde-fou-pilote_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Runner de lecture filtrée avec garde-fou pilote (automation × base de tâches)

Date : 17.07.2026 · Portée : générique — applicable à tout outil d'automatisation lisant une base de tâches partagée humains/agents (Notion, Airtable, Jira…) pour n'en traiter qu'un périmètre strictement borné pendant une phase pilote. Aucune référence à des IDs spécifiques.

---

## Principe

Quand un automate commence à travailler dans une base partagée avec des humains, son premier périmètre doit être **une seule cible désignée**, verrouillée par un **garde-fou codé en dur** — pas un simple filtre de vue. Le runner lit large (la base), mais un nœud de code ne laisse passer que l'intersection : `ID cible + marqueur agent + statut prêt`. On élargit le périmètre en changeant UNE constante, après validation humaine.

## Marche à suivre

1. **Partir du workflow de test de connexion** (créé lors de la mise en place du token) : le renommer d'après sa fonction, renommer chaque nœud en verbe d'action lisible (« Read agent tasks », « Select and map the pilot task ») — le canvas doit se lire comme une phrase.
2. **Lecture large** : nœud « lire la base » (par ID, sortie simplifiée). Pas de filtre côté source au MVP — le filtre de sécurité vit dans le code, visible et versionnable.
3. **Nœud garde-fou + mapping** (code) :
   - constantes en tête : `PILOT_ID`, marqueur agent, statut « prêt » ;
   - filtre : ne garder que l'élément qui satisfait LES TROIS conditions ;
   - mapping : projeter les champs bruts en un brief propre (id, titre, catégorie, relation principale, notes, statut) — le reste de la chaîne ne voit que ce contrat.
4. **Prouver le garde-fou dans les 3 sens** :
   a. cible NON prête → sortie vide ;
   b. cible prête → elle sort SEULE, correctement mappée ;
   c. **leurre** (élément conforme à tous les critères SAUF l'ID) créé exprès → il est ignoré, puis supprimé. C'est ce test qui prouve le cloisonnement.
5. **Filets** : brancher le workflow d'erreur global (alerting), poser les tags/l'emplacement de rangement, laisser le déclencheur MANUEL tant que la boucle complète n'est pas validée.
6. **Documenter** : le contrat de sortie du mapping (les chantiers suivants s'y branchent), l'emplacement du garde-fou et la règle d'élargissement (qui change la constante, quand).

## Garde-fous

- Le périmètre de sécurité vit dans le CODE, pas dans une vue/un filtre modifiable par l'UI.
- Tester avec les données réelles ET un leurre — jamais seulement le chemin heureux.
- Après modification des données sources, re-exécuter le WORKFLOW ENTIER : « exécuter le nœud » rejoue l'input en cache, pas un nouveau fetch.
- Ne pas publier/planifier le runner tant qu'il ne fait que lire — le déclencheur automatique vient après la validation de la boucle d'écriture.

## Pièges d'exécution (navigateur piloté)

- Éditeur de code web (CodeMirror) : l'auto-fermeture des accolades laisse des lignes orphelines EN FIN de code lors d'une frappe longue → toujours scroller à la fin et purger avant d'exécuter.
- Renommage de nœud : sélection + **F2** (le double-clic zoome le canvas).
- Entrées de menus : localiser par arbre d'accessibilité, pas aux coordonnées.

---

## Difficultés rencontrées

1. SyntaxError fantôme due aux accolades auto-fermées ajoutées en fin de code par l'éditeur.
2. Faux négatif de test : « exécuter le nœud » réutilisait l'input en cache d'avant le changement de statut de la cible.
3. UI par moments récalcitrante (double-clic = zoom, menus qui se referment, onglet neuf impossible à piloter).

## Solutions implémentées

1. Contrôle systématique de la fin du code (cmd+Down) + purge des orphelines avant exécution.
2. Règle : données changées → exécution du workflow entier ; exécution de nœud = debug sur input connu uniquement.
3. F2 pour renommer, accessibilité pour les menus, geste manuel délégué à l'humain quand l'onglet résiste (2 clics vs 10 min de lutte).

## Lessons learned

1. **Un périmètre pilote n'est crédible que testé au leurre** — le chemin heureux ne prouve rien sur ce que l'automate NE doit PAS toucher.
2. **Le contrat de mapping est une interface** : le figer tôt (et le documenter) découple la lecture des chantiers d'exécution qui suivent.
3. Élargir un périmètre doit coûter UN changement de constante + UNE validation humaine — si c'est plus compliqué, le design du garde-fou est mauvais ; si c'est moins, il n'y a pas de garde-fou.
