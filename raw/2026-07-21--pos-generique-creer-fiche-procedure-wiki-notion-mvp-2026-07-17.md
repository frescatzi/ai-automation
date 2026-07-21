---
type: raw
title: "POS-GENERIQUE_creer-fiche-procedure-wiki-notion-mvp_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_creer-fiche-procedure-wiki-notion-mvp_2026-07-17.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE — Créer une fiche de procédure (SOP) dans un wiki Notion, format MVP

Date : 17.07.2026
Type : POS-GÉNÉRIQUE (réutilisable, dépouillé des IDs spécifiques).
But : ajouter une fiche de procédure courte et actionnable dans un wiki Notion « base de connaissances », en référençant les bases et vues existantes sans les copier.

## Pré-requis
- Un wiki Notion (page contenant une base wiki) avec au moins une propriété Category (ex : SOP / How-to / Reference) et un Summary.
- Le connecteur Notion opérationnel (le vérifier en début de session par un simple fetch).

## Étapes
1. Vérifier le connecteur (fetch de la page wiki).
2. Lire le schéma des bases que la fiche va référencer (statuts, champs, template) pour décrire le système RÉEL, pas un état supposé. Lire une entrée existante du wiki comme modèle de format et du principe « référencer, pas copier ».
3. Créer la fiche par API : `create-pages` avec parent = data source du wiki. Renseigner Name, Category, Summary. Contenu au format MVP :
   - When to use
   - Where it lives (lien vers la page/vue réelle)
   - Numbered steps (courtes, concrètes)
   - Common mistakes
   - Owner
   - Related (liens vers les fiches sœurs)
4. RÉFÉRENCER par lien, jamais incruster la base. Une base ou une vue incrustée réapparaît dans la sidebar SOUS la fiche (loi sidebar Notion) : préférer les mentions et les liens.
5. Montrer à la personne, corriger par petits search-and-replace ciblés (`update_content`), faire valider, documenter.

## Bonnes pratiques de correction
- Re-fetch la page juste avant toute correction (surtout si la personne édite en parallèle au navigateur).
- Utiliser `update_content` (chirurgical) plutôt que `replace_content` (qui écrase tout).
- Les liens par ID survivent au renommage d'une page : ne mettre à jour que le texte visible.

## Convention « Related = titres cliquables »
Dans le bloc Related, transformer chaque titre de fiche recommandée en LIEN direct vers sa page, pour amener l'utilisateur droit au but (pas juste un nom en texte). Câbler dans les deux sens dès que les fiches cibles existent. Un lien vers l'index du wiki peut compléter.

## Style
Zéro tiret cadratin « — » : utiliser deux-points, point-virgule ou parenthèses.

## Difficultés rencontrées
- Édition concurrente de la même page (navigateur) pendant l'écriture par API : les search-and-replace échouent (texte réel différent), et des renommages structurels peuvent survenir en cours de route.
- `update_content` exige un match exact de la chaîne.

## Solutions implémentées
- Re-fetch avant chaque écriture ; vérifier la vérité terrain (fetch des bases concernées) avant d'aligner le vocabulaire.
- Éditions ciblées et incrémentales plutôt qu'un remplacement global.

## Lessons learned
- Décrire le système réel : lire les schémas des bases d'abord.
- Référencer sans copier : une seule vérité, et une sidebar propre.
- Les liens par ID sont robustes au renommage.
- Related en titres-liens cliquables = navigation directe pour l'utilisateur.
- Pas de tiret cadratin dans les livrables.
