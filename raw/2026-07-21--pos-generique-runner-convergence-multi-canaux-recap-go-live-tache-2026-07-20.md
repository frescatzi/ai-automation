---
type: raw
title: "POS-GENERIQUE_runner-convergence-multi-canaux-recap-go-live-tache_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-convergence-multi-canaux-recap-go-live-tache_2026-07-20.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Runner de convergence multi-canaux → récap « go-live » dans une tâche

Patron réutilisable : détecter que plusieurs livrables indépendants (canaux) d'une même unité (vague) ont tous franchi un seuil d'approbation, puis assembler un récap consolidé « prêt à agir » dans une tâche dédiée, sans jamais déclencher l'action finale (l'humain garde le gate).

## Difficultés
1. **Convergence = ET logique sur des sources hétérogènes.** Les canaux vivent dans des bases différentes, avec des statuts d'approbation différents (Newsletter : Approved/Scheduled/Sent ; Instagram : Approved/Scheduled/Posted ; page web : Approved/Live). Il faut un « tous verts » sur des vocabulaires distincts.
2. **Granularités mélangées.** Certains livrables sont par sous-unité (par vague), d'autres par unité mère (une page événement par édition, pas par vague). Un « tous les canaux de la vague » naïf casse sur la source par-édition.
3. **Où déposer un artefact « méta »** qui n'est pas un canal de plus mais la synthèse des canaux, sans créer une place de plus à surveiller ni dupliquer les corps déjà rédigés ailleurs.
4. **Rétroactivité.** Le générateur de tâches (amont) ne repasse pas sur les unités déjà créées : ajouter un type de tâche ne le pose pas sur l'existant.
5. **Éviter tout déclenchement d'action** alors qu'on lit des statuts « Sent »/« Posted » qui pourraient laisser croire qu'il faut ré-agir.

## Solutions
1. **Un filtre HTTP par source, chacun sur SON vocabulaire post-approbation** ; le runner ne voit que des lignes déjà approuvées. « Présent dans le jeu Approved+ » = « approuvé », pas besoin de relire le statut.
2. **Indexer par la bonne clé selon la granularité** : les sources par sous-unité par `unité||sous-unité` (issu du titre canon), la source par unité mère par `unité` seule. La convergence exige la présence de chaque source requise dans son index. La précondition « unité mère » (page événement) est une condition globale de la sous-unité.
3. **Déposer le récap dans une tâche dédiée** posée par le générateur amont, consommée comme les autres (rails To Do → To Validate, auto-drainage par le filtre To Do). Le récap **pointe** vers les tâches canaux pour les corps complets au lieu de les recopier (une seule source de vérité, Notes courtes).
4. **Backfill ponctuel, pas de mécanisme permanent** : on ajoute le type de tâche pour le go-forward, et on crée une fois à la main les tâches manquantes des unités en cours qui le méritent (nommage cohérent, source réelle). On documente précisément ce qui a été exclu (restes de test, noms incohérents) plutôt que de tout créer aveuglément.
5. **Le runner n'écrit que Notes + statut To Validate**, aucun appel d'action. Un statut « Sent/Posted » côté source ne fait que confirmer la convergence, il ne relance rien.

## Lessons
- **Convergence = jointure, pas séquence.** On ne chaîne pas les canaux ; on les lit en parallèle et on teste un ET sur des index. Ajouter/retirer un canal = ajouter/retirer une source dans le ET.
- **Un artefact de synthèse doit référencer, pas recopier.** Le récap go-live gagne à pointer vers les livrables déjà prêts (tâches canaux remplies) : Notes courtes, zéro divergence de contenu, sous la limite Notion 2000 car d'un rich_text.
- **Distinguer clé de matching (titre) et donnée d'affichage (relation).** Le matching reste par titre (relation non fiable en scan) ; la relation ne sert qu'à fabriquer un lien de navigation (ici, lien Notion vers la page mère où vit l'URL externe), usage d'affichage sans dépendance de matching.
- **Précondition globale explicite.** « Prêt à diffuser » implique que la cible de diffusion (page événement) est en ligne : en faire une condition dure évite un récap qui pousse vers une page non publiée.
- **Rétroactivité : nommer ce qu'on ne fait pas.** Ajouter un type de tâche est go-forward par nature ; le backfill est un geste ponctuel, borné, et surtout journalisé (ce qui est créé, ce qui est écarté et pourquoi) pour ne pas polluer avec des tâches qui ne convergeront jamais.
