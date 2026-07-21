---
type: raw
title: "POS-GENERIQUE_ecriture-controlee-statuts-agent-jamais-done_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_ecriture-controlee-statuts-agent-jamais-done_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Écriture contrôlée d'un automate dans une base de tâches (claim + report, jamais l'état final)

Date : 17.07.2026 · Portée : générique — tout automate qui met à jour des tâches dans une base partagée avec des humains (Notion, Airtable, Jira…), sous la règle « l'automate propose, l'humain dispose ». Aucune référence à des IDs spécifiques.

---

## Principe

Un automate qui écrit dans une base de tâches suit un contrat en deux gestes : **claim** (je prends la tâche → statut « en cours ») puis **report** (je dépose mon résultat → champ de compte-rendu + statut « à valider »). **L'état final (« terminé ») n'existe nulle part dans sa configuration** : la clôture appartient à l'humain. C'est un invariant STRUCTUREL, pas une consigne — l'outil est physiquement incapable d'écrire l'état qu'il ne connaît pas.

## Marche à suivre

1. **Claim** : nœud « mettre à jour » → statut = « en cours ». La cible vient du brief mappé en amont (jamais d'ID en dur ici — c'est le garde-fou amont qui borne le périmètre).
2. **Report** : second nœud « mettre à jour » → champ compte-rendu = résultat du travail + statut = « à valider ». Au MVP, le résultat est un placeholder ; le vrai moteur (agent/LLM) s'intercale entre les deux ensuite.
3. **Référencer la cible défensivement** : l'identifiant peut arriver sous deux noms selon le chemin (sortie du nœud précédent vs mapping d'origine) → expression `id_sortie || id_brief`.
4. **Écritures dynamiques** : si l'outil ne peut pas charger la liste des propriétés (cible en expression), donner la clé au format `nom|type` en expression — le champ de valeur correspondant apparaît.
5. **Tester bout-en-bout sur données réelles** : remettre la cible à l'état initial, exécuter le WORKFLOW ENTIER, puis vérifier les 3 effets : statut final correct · compte-rendu écrit · la tâche apparaît dans la file de validation de l'humain.
6. **Échec d'exécution** (prévu pour l'étape moteur) : statut = « bloqué » + raison dans le compte-rendu — jamais silencieux, jamais « terminé ».

## Garde-fous

- L'état « terminé » n'apparaît dans AUCUN nœud, aucune constante, aucune expression de l'automate.
- Anti double-run : l'entrée n'accepte que le statut « prêt » — une tâche déjà en cours n'est jamais re-saisie.
- Un seul écrivain automate par tâche à la fois (le claim marque la possession).
- Vérifier le credential de CHAQUE nouveau nœud (l'auto-assignation prend le premier compatible).

## Pièges d'exécution (constatés)

- Les parseurs d'URL des nœuds intégrés ne reconnaissent pas tous les formats d'URL de l'outil cible — construire l'URL canonique depuis l'ID est plus fiable que passer l'URL reçue.
- Un nœud amont DÉSACTIVÉ (clic parasite) laisse passer les données inchangées : l'aval reçoit un autre contrat et les expressions se vident — lire le bandeau d'état du panneau d'input avant de chercher ailleurs.
- Les listes d'options (statuts, propriétés) ne chargent pas quand la cible est dynamique — le mode expression `nom|type` est le contournement standard.

---

## Difficultés rencontrées

1. URL de l'API non reconnue par le nœud de mise à jour ; sélecteur de mode récalcitrant à l'automatisation.
2. Listes de propriétés/options en erreur dès que la page cible est une expression.
3. Nœud désactivé par mégarde → symptôme trompeur en aval (expression vide).

## Solutions implémentées

1. URL canonique construite depuis l'ID en expression, avec fallback `||` sur les deux noms possibles de l'identifiant.
2. Clés en mode expression au format `nom|type` → champs de valeur rendus normalement.
3. Diagnostic par le panneau d'input (état du nœud amont) avant toute modification.

## Lessons learned

1. **La sécurité par absence** : ce que l'automate ne peut pas exprimer, il ne peut pas le faire — retirer l'état final de son vocabulaire vaut tous les garde-fous logiques.
2. **Claim/report est un protocole social autant que technique** : l'humain voit qui travaille (en cours) et où valider (à valider) sans qu'on lui explique rien — les vues existantes font l'interface.
3. Tester la boucle avec un placeholder AVANT de brancher le moteur : on prouve la plomberie seule, et le branchement du moteur ne peut plus casser que le moteur.
