---
type: raw
title: "POS-EXACT_LUMINA-demenagement-espace-notion-et-renommage-token_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_lumina-demenagement-espace-notion-et-renommage-token_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Déménagement LUMINA dans son teamspace Notion + renommage du token

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) / LUMINA AI OS · Espaces : Notion (workspace « AFTRSN ») + n8n LUMINA OS
Rôle : marche à suivre exacte du chantier « ranger tout LUMINA dans son espace + renommer/repointer le token » (spec `SPEC-BUILD_LUMINA-demenagement-espace-notion-et-repointage-token_2026-07-17.md`). **Validé par Karter le 17.07.2026.**

---

## 0. Résultat

Toute la connaissance LUMINA vit désormais dans le teamspace **`LUMINA AI OS - HQ`** (`38224f2f-ef0c-81f9-9080-0042b67d3909`) ; le token Notion du pipeline LUMINA est renommé **`LUMINA-n8n-Notion`** (connexion Notion + credential n8n). Aucun workflow recâblé (renommage ID-stable). Les anciennes pages AFTRSN (ancienne « The Backstage (not in use) ») sont déclarées inutiles → supprimées par Karter.

## 1. Découvertes de cadrage (ont simplifié le chantier)

1. **Un connecteur MCP = UN workspace, mais PLUSIEURS teamspaces.** Vérifié : avec la même connexion, lecture de pages dans `AFTER SUN PEOPLE - HQ`, dans l'ancienne Backstage ET dans `LUMINA AI OS - HQ`. Le workspace connecté est « AFTRSN » (`cfc24f2f-ef0c-810f-9fc5-0003d64e4d5f`). Tout le contenu vit dans ce workspace unique → pas de contrainte multi-workspace. (Le 404 initial sur une page fraîche = simple partage non fait, pas une limite de teamspace.)
2. **« Tout LUMINA » = UNE seule base.** Les pages système (LUMINA Système multi-agents, Système de connaissance, les SOP Lumina, concepts…) ne sont pas éparpillées : ce sont les **lignes de la base `AI Automation — Knowledge`** (`6fac8cb8-91e4-4c74-9e37-a86419826905`, data source `657827e3-1e2a-48d5-8f52-5954e3618687`), le wiki publié par le pipeline intake→GitHub→Notion. Chaque ligne a un `Source` = chemin GitHub. → **Déplacer cette seule base = déplacer tout LUMINA.**
3. Destination : choix passé d'une nouvelle page → l'ancienne Backstage → finalement le **teamspace existant `LUMINA AI OS - HQ`** (le plus propre : déjà un vrai teamspace dédié).

## 2. Déplacement (ID-stable)

- Cible : base `AI Automation — Knowledge` (`6fac8cb8…`), initialement sous « Teamspace Home » (`ab824f2f…`) du teamspace AI-AUTOMATION.
- **Tentative API échouée** : `notion-move-pages` avec `new_parent = {page_id: <ID teamspace LUMINA>}` → `400 « Could not load new parent … or missing edit permission »`. Un teamspace n'est pas une page-parent adressable par l'API (fetch du teamspace = 404), et l'intégration n'avait pas de droit d'écriture au niveau racine du teamspace.
- **Contournement retenu** : Karter (propriétaire) a fait le **drag dans la sidebar Notion** — déplacement entre teamspaces instantané. 2 s vs lutte API.
- **Vérification post-move** : `notion-fetch` de la base → `<ancestor-path>` désormais VIDE (plus sous « Teamspace Home », passée au niveau racine du teamspace LUMINA). **IDs inchangés** : base `6fac8cb8…`, data source `657827e3…`, schéma et lignes intacts.

## 3. Ciblage ≠ Accès (le point à ne pas rater)

Deux choses distinctes côté n8n après un déplacement Notion :
- **Ciblage** : le node n8n pointe vers l'**ID** de la base. L'ID ne change pas au déplacement → ciblage intact. ✅
- **Accès** : le droit de l'intégration à lire/écrire la base à son nouvel emplacement. Un déplacement **entre teamspaces** peut le casser SI l'intégration est scopée par teamspace. Ici : accès du token au **niveau workspace** → conservé (Karter confirmé). Filet : si un run futur renvoie une erreur d'accès Notion, rajouter l'accès de l'intégration au teamspace `LUMINA AI OS - HQ`.

## 4. Renommage du token (geste Karter — secrets)

- **Connexion Notion** `AFTRN-n8n` → **`LUMINA-n8n-Notion`**.
- **Credential n8n** « Notion account » → **`LUMINA-n8n-Notion`**.
- **Renommage ID-stable** : l'ID du credential ne change pas → aucun workflow à recâbler, tous continuent de pointer dessus.
- Motivation : `AFTRN-n8n` était trompeur (« AFTRN » évoquait AFTRSN alors que c'est le pipeline LUMINA) ; « Notion account » était générique. Désambiguïsation permise par le fait que le Backstage AFTRSN a désormais son propre credential dédié `AFTRSN-n8n-Backstage` (Phase 2).

## 5. Invariants tenus

- **Credential Backstage `AFTRSN-n8n-Backstage` jamais touché** (isolé, propre).
- **Additif avant destructif** : contenu déplacé et vérifié AVANT tout renommage ; suppressions (anciennes pages AFTRSN) laissées à Karter, en dernier.
- **Secrets/tokens par Karter uniquement** : renommage connexion + credential fait par lui.
- **Suppressions par Karter uniquement** : l'assistant ne supprime pas (geste irréversible une fois la corbeille vidée).

---

## Difficultés rencontrées

1. **Confusion d'appartenance teamspace** : des pages dont l'ID commence par `38224f2f` (préfixe du teamspace LUMINA) vivaient en fait sous l'ancienne Backstage — le préfixe d'ID n'indique PAS le teamspace réel ; seul l'`ancestor-path` fait foi.
2. **API incapable de déplacer vers la racine d'un teamspace** (parent non chargeable / droit d'écriture manquant).
3. **Fausse assurance possible** « ciblage OK = tout OK » : le ciblage par ID survit au move, mais l'accès de l'intégration est un sujet séparé.

## Solutions implémentées

1. Vérifier l'`ancestor-path` réel de chaque page (pas se fier au préfixe d'ID) pour savoir où vit vraiment un objet.
2. Déléguer le déplacement au drag humain (propriétaire) quand l'API bute sur la racine d'un teamspace — plus rapide que de forcer.
3. Distinguer explicitement ciblage (ID, stable) et accès (partage d'intégration, sensible au teamspace) ; documenter le filet (rajouter l'accès si un run casse).

## Lessons learned

1. **Auditer AVANT de supposer l'éparpillement** : « tout X » s'est révélé être les lignes d'UNE base. Déplacer le conteneur déplace tout, en un geste — encore fallait-il le voir.
2. **Un connecteur MCP Notion = un workspace, mais multi-teamspaces** : ne pas se sur-contraindre. La seule vraie frontière est le workspace.
3. **Déplacer/renommer dans Notion est ID-stable** : les automations qui ciblent par ID survivent au déplacement d'une base et au renommage d'un credential — mais **l'accès d'une intégration peut changer en franchissant un teamspace**. Ciblage ≠ accès.
4. **Quand l'API bute, le geste humain de 2 s gagne** : un drag de propriétaire dans la sidebar bat une lutte contre un parent non adressable.
