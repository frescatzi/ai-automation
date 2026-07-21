---
type: raw
title: "POS-GENERIQUE_brancher-moteur-agent-boucle-claim-report_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_brancher-moteur-agent-boucle-claim-report_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Intercaler un moteur agent (LLM) dans une boucle claim/report et valider la boucle entière

Date : 17.07.2026 · Portée : générique — tout automate qui, dans une base de tâches partagée humains/agents, doit faire exécuter le travail par un moteur (agent LLM, sous-workflow, service) puis déposer le résultat pour validation humaine. Aucune référence à des IDs spécifiques.

---

## Principe

Une fois la plomberie claim/report prouvée avec un placeholder (voir POS-GÉNÉRIQUE « écriture contrôlée »), on **intercale le moteur entre claim et report** : claim (statut « en cours ») → moteur (produit le travail + un verdict) → report (compte-rendu réel + statut « à valider » OU « bloqué »). Le moteur ne clôture jamais : il propose. L'invariant « pas d'état terminé dans l'automate » reste câblé.

## Marche à suivre

1. **Connaître le contrat du moteur** avant de câbler : quel champ d'entrée il accepte (souvent un seul, ex. `message` — les autres sont filtrés) et quelle forme de sortie il renvoie (ex. `{ texte, verdict }` avec `verdict ∈ succès|échec|timeout`). Construire le brief d'entrée à partir des données de la tâche mappée en amont.
2. **Insérer le nœud moteur** entre claim et report. Le brief référence le nœud de mapping (titre, notes…), pas la sortie du claim (qui ne porte que l'identifiant/statut).
3. **Porter la logique succès/échec dans le nœud report, pas dans un IF** : deux expressions conditionnelles suffisent —
   - statut = `verdict == succès ? « à valider » : « bloqué »` ;
   - compte-rendu = `(verdict == succès ? "" : "[BLOQUÉ] ") + texte_du_moteur`.
   Moins de nœuds = moins de surface de casse (un IF se désactive/se dé-câble facilement en édition).
4. **Référencer les données au bon endroit** : la sortie directe du moteur est l'`input` du nœud report → y accéder via la variable d'item courant (`$json`-équivalent), court et sans échappement. Ne référencer un nœud par son nom que pour ce que l'item courant ne porte pas (ex. l'identifiant de la cible, qui vient du mapping). Éviter de référencer par nom un nœud dont le nom contient des caractères spéciaux (guillemets) — préférer l'item courant.
5. **Tester la boucle ENTIÈRE sur données réelles** : remettre la cible à l'état « prêt », exécuter le workflow complet (pas le nœud seul — il rejoue l'input en cache), attendre le vrai temps de calcul du moteur, puis vérifier les 3 effets : statut = « à valider », compte-rendu = sortie réelle du moteur, la tâche apparaît dans la file de validation humaine.
6. **Laisser l'humain clôturer** : la validation (passage à « terminé ») est un geste humain hors automate — c'est le test final de la boucle, et la preuve que l'invariant tient.
7. **Relire et valider par API après un build visuel** : si un accès API à l'outil d'automatisation existe, lire la configuration réelle et lancer une validation statique — c'est le seul moyen de voir les défauts tolérés à l'exécution mais présents dans la config (artefacts d'éditeur, littéraux parasites).

## Garde-fous

- Aucune expression, constante ou option de l'automate ne contient l'état « terminé ».
- Échec/timeout du moteur → statut « bloqué » + raison ; jamais silencieux, jamais « terminé ».
- Le brief d'entrée du moteur est construit depuis la tâche mappée (contrat amont borné), pas depuis des IDs en dur au niveau du moteur.
- Un run réel dépend d'un vrai temps de calcul (LLM) : distinguer « en cours » (long) de « en erreur ».
- Ne pas re-exécuter pour un correctif cosmétique déjà prouvé inoffensif par validation statique, si cela détruit un état validé par l'humain.

## Pièges d'exécution (constatés)

- Après insertion du moteur, **l'input du nœud report change de contrat** (c'est la sortie du moteur, plus la tâche) : les expressions qui supposaient l'ancien input se vident ou pointent au mauvais endroit → re-vérifier chaque référence.
- **Éditeurs de code web (auto-fermeture)** : guillemets/parenthèses/accolades se doublent ; un délimiteur surnuméraire en fin d'expression peut être toléré à l'exécution (concaténé comme littéral) tout en étant un défaut — invisible sans relire la config brute.
- Nom de nœud à caractères spéciaux (guillemet simple) : casse les références par nom → préférer l'item courant.

---

## Difficultés rencontrées

1. Changement de contrat d'input au nœud report après insertion du moteur.
2. Références par nom fragiles quand le nom du nœud moteur contient un guillemet.
3. Délimiteur surnuméraire hérité d'un éditeur visuel, toléré à l'exécution donc passé inaperçu.

## Solutions implémentées

1. Séparer les sources : identifiant/cible depuis le mapping amont (par nom) ; verdict/texte depuis l'item courant (sortie directe du moteur).
2. Utiliser l'item courant plutôt que le nom du nœud moteur → zéro échappement.
3. Correctif chirurgical par API (find/replace sur le champ) + validation statique, une fois un accès écriture disponible ; à défaut, relire la fin des expressions dans l'éditeur.

## Lessons learned

1. **Intercaler un moteur ne change pas l'invariant, il le remplit** : le placeholder devient un vrai compte-rendu, mais la règle « propose, ne clôt pas » et la mécanique claim→report restent identiques. Prouver la plomberie AVANT le moteur permet d'isoler toute panne au moteur seul.
2. **Une ternaire vaut mieux qu'un IF** pour router succès/échec vers deux statuts : moins de nœuds, expression versionnable, rien à re-câbler.
3. **La sortie directe d'un nœud est l'input du suivant** — la référencer via l'item courant est la voie la plus robuste et la plus lisible ; réserver les références par nom aux données absentes de l'item courant.
4. **La validation statique par API est le filet qui rattrape ce que l'exécution tolère** : un build visuel qui « marche » n'est pas un build propre — relire la config réelle est un réflexe, pas une option.
