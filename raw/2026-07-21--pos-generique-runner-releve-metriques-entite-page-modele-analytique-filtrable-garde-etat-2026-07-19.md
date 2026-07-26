---
type: raw
title: "POS-GENERIQUE_runner-releve-metriques-entite-page-modele-analytique-filtrable-garde-etat_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-releve-metriques-entite-page-modele-analytique-filtrable-garde-etat_2026-07-19.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-GÉNÉRIQUE · Runner de relève de métriques d'une ENTITÉ (page/objet) : modèle analytique filtrable + garde d'état + double source

**Date :** 19.07.2026 · **Contexte d'origine :** AFTRSN P3 · M5c Website Metrics · **Type :** POS-GÉNÉRIQUE (réutilisable).

> Complément du POS-GÉNÉRIQUE « relève de métriques plateforme → base Metrics par canal » (M5b). Ce patron-ci traite le cas où les chiffres concernent **une entité précise** (une page, un objet, un événement) et non une campagne : il faut **filtrer la métrique par l'identifiant de l'entité**, ne relever que **tant que l'entité est dans le bon état**, et parfois **croiser deux sources** (une mesure analytique + un comptage transactionnel).

---

## 1. Difficultés
1. **Isoler la métrique d'UNE entité** : les API « analytics » globales renvoient le total du site/compte, sans filtre par page/objet — inutilisable pour une ligne par entité.
2. **Savoir quand relever** : une entité a un cycle de vie (brouillon → publié → terminé → annulé) ; relever un brouillon n'a pas de sens, relever indéfiniment gaspille.
3. **Deux chiffres, deux API** : une même ligne Metrics agrège souvent une mesure de trafic ET un comptage transactionnel (inscriptions, ventes) qui vivent dans des services différents.
4. **Rétention limitée** des données analytiques : la fenêtre « depuis la mise en ligne » peut dépasser ce que la plateforme conserve.
5. **Retrouver l'identifiant technique** de l'entité (slug, id) qui sert de clé de filtre, sans le stocker partout.

## 2. Solutions (le patron)
1. **Chercher un modèle analytique FILTRABLE avant de coder.** Si l'API « données » simple ne filtre pas par entité, chercher une couche « modèle sémantique / dimensions » : lister les modèles, inspecter le schéma du bon modèle (mesures + **dimensions**), repérer la dimension d'identité (ex. `page_path`, `product_id`) qui accepte un filtre `EQUAL`/`START_WITH`. Prouver le filtre en direct sur une entité réelle **avant** le build (une requête = la preuve).
2. **Garde d'état lue à la source, pas devinée.** Avant de relever, **lire l'objet** (un GET) pour obtenir son état réel (publié ? annulé ?) ET son identifiant de filtre (slug). Règle simple et robuste : relever si l'objet est **publié** (présence d'un `publishedDate`/équivalent, plus fiable qu'un flag « draft ») **et non annulé**. Un objet « terminé » reste souvent pertinent (trafic résiduel). Sinon, skip.
3. **Double source assumée dans la même itération** : chaîner mesure analytique (trafic) puis comptage transactionnel (inscriptions), chacun avec sa clé (chemin de page vs id d'objet), et fusionner en une ligne Metrics à l'upsert. Un comptage à 0 pour un mode d'inscription externe n'est pas une erreur — c'est la réalité.
4. **Fenêtre « depuis la mise en ligne », plafonnée à la rétention** : `start = max(dateDeMiseEnLigne, aujourd'hui − N)` où N = fenêtre de conservation de la plateforme ; `end = aujourd'hui`. Chiffre **cumulé** rafraîchi chaque jour (cohérent avec un compteur qui grandit).
5. **Upsert par TITRE + relations à la création** (identique au patron M5b) : la ligne Metrics porte le titre de la ligne de contenu de l'entité ; find-par-titre → update sinon create ; relations `Content` + entité posées d'emblée (navigation bidirectionnelle). 1 entité = 1 ligne datée (`As-of`).
6. **Écritures Notion en HTTP brut** pour number/date (JSON construit en nœud Code). **Garde-fou de drainage dans une boucle** : la branche « entité inactive » doit **reboucler** vers le nœud de boucle (sinon l'itération se bloque). **Tester l'idempotence sur le 2ᵉ passage.**

## 3. Lessons
- **La bonne API n'est pas toujours la première trouvée** : l'API « data » évidente peut être trop grossière (site entier). La couche « modèle sémantique » filtrable existe souvent à côté — la chercher AVANT de renoncer au niveau de détail demandé.
- **Lire l'état d'une entité vaut mieux que le supposer** : un GET sur l'objet donne à la fois la garde (publié/annulé) et la clé de filtre (slug) — un seul appel sert deux besoins.
- **Publié = présence d'une date de publication**, pas l'absence d'un flag brouillon : c'est le test le plus fiable pour « ne pas relever un draft ».
- **La rétention est une contrainte de fenêtre, pas un bug** : plafonner le début de fenêtre à la durée de conservation, et le documenter, évite les erreurs « pas de données pour cette date ».
- **Prouver chaque endpoint en direct avant le build** (filtre par entité, comptage) transforme le build en simple assemblage : ici les trois appels (visites filtrées, comptage RSVP, lecture d'événement) étaient prouvés avant d'écrire le premier nœud.
