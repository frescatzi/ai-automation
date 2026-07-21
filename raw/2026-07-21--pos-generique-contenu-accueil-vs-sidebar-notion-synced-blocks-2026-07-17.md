---
type: raw
title: "POS-GENERIQUE_contenu-accueil-vs-sidebar-notion-synced-blocks_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_contenu-accueil-vs-sidebar-notion-synced-blocks_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Notion : arbitrer « contenu affiché sur une page d'accueil » vs « sidebar minimale » (+ patterns synced blocks)

Date : 17.07.2026 · Portée : générique, applicable à tout espace Notion (aucun ID spécifique)

## Le problème

On veut souvent deux choses à la fois sur un portail/page d'accueil Notion : (1) afficher un tableau de bord vivant (bases, vues filtrées) directement sur la page, et (2) garder un menu latéral (sidebar) minimal qui ne liste que les grandes zones. Ces deux exigences sont **structurellement incompatibles**.

## La loi à connaître

**Toute base ou vue liée VISIBLE sur une page apparaît dans la sidebar comme enfant de cette page.** Vérifié empiriquement dans tous les cas de contournement plausibles :

- bloc base/linked view posé directement sur la page → listé ;
- bloc DANS une colonne ou un callout → listé ;
- bloc DANS un toggle heading (même replié) → listé ;
- **synced block reference** (miroir synchronisé d'un bloc vivant ailleurs) → les bases qu'il rend sont listées AUSSI sous la page hôte du miroir (double apparition : sous la page d'origine ET sous la page miroir).

Il n'existe aucun réglage Notion pour masquer une base de la sidebar. Le seul curseur est : **où poser physiquement (ou refléter) le contenu**.

## L'arbitrage à présenter à l'utilisateur

1. **Le contenu gagne** : le dashboard s'affiche sur l'accueil ; la sidebar liste ses vues sous l'accueil (les nommer proprement avec emoji pour que ce soit lisible).
2. **La sidebar gagne** : le dashboard vit dans une sous-page d'une zone ; l'accueil offre une porte (1 clic). Sidebar minimale garantie.

Présenter l'incompatibilité AVANT d'exécuter, avec preuve si besoin (test sur page brouillon) — l'utilisateur choisit en connaissance de cause.

## Patterns synced blocks découverts (utiles quand la pollution sidebar est acceptable)

- **Créer une linked view DANS un synced block : navigateur uniquement** (`/synced` puis, à l'intérieur, `/linked view of data source` ; on peut recopier la config d'une vue existante en la choisissant dans « Views on <base> »). L'API `create-view` ne sait viser que la fin d'une page.
- **Envelopper des blocs existants dans un synced block : API OK** — `update-page replace_content` en nichant les blocs `<database url=…>` existants dans `<synced_block>…</synced_block>`.
- **Déballer un synced block : PAS par replace_content** (le garde-fou croit à une suppression des bases) → **`move-pages` des blocs vers le MÊME parent** les sort du synced block proprement ; le synced block vidé se supprime ensuite par replace_content.
- La **reference** (`<synced_block_reference url=…>`) rend le contenu pleinement interactif (édition, création, changement de statut) — c'est un vrai miroir, pas une copie.

## Pièges de renommage associés (rencontrés en réparant l'espace)

- Renommer « au mauvais endroit » un wrapper de linked view peut renommer **la data source partagée** (donc la base d'origine partout). Symptômes : doublons dans la sidebar, base introuvable sous son vrai nom.
- Le **header affiché d'une base inline** est un override de bloc, distinct du nom de la data source : après réparation du nom par API (`update-data-source title`), vérifier le header dans l'UI et le corriger à la main si besoin (double-clic sur le titre → retaper le nom).
- Diagnostic rapide des dégâts : `notion-fetch` sur chaque `collection://…` suspecte et comparer `name` au nom attendu ; les relations/rollups survivent au renommage (seul le nom change).

## Difficultés rencontrées

- L'incompatibilité contenu/sidebar n'est documentée nulle part par Notion — elle ne se découvre qu'en testant.
- Les tests navigateur sur Notion sont fragiles : scroll capricieux (boards à scroll horizontal), état « still loading » persistant qui bloque les outils à injection de script.

## Solutions implémentées

- Protocole de test à 3 étages sur page brouillon : (1) le bloc se crée-t-il ? (2) se rend-il ailleurs via reference ? (3) la sidebar reste-t-elle propre ? — 10 minutes qui évitent une architecture mort-née.
- Contournements navigateur : `cmd+Down` (fin de page), repli des sections de la sidebar pour tout voir sans scroller, zoom de région pour lire les petits éléments.

## Lessons learned

- **Tester la faisabilité sur brouillon avant de toucher la structure réelle** — surtout quand l'exigence combine affichage et navigation.
- Quand deux exigences utilisateur se contredisent techniquement, le livrable n'est pas un compromis silencieux : c'est **la preuve de l'incompatibilité + un choix explicite**.
- Après toute session de modifications manuelles par l'utilisateur, **re-lire l'état réel** (fetch) avant d'appliquer quoi que ce soit : la doc décrit hier, pas maintenant.
