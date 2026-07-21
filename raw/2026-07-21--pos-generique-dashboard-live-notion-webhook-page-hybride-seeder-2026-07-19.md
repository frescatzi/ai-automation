---
type: raw
title: "POS-GENERIQUE_dashboard-live-notion-webhook-page-hybride-seeder_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_dashboard-live-notion-webhook-page-hybride-seeder_2026-07-19.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Dashboard vivant « dans Notion » via webhook hébergé (page hybride) + seeder de placeholders

Patron réutilisable, tiré de M8 (AFTRSN P3). But : afficher un tableau de bord **à jour et interactif** adossé à une base Notion, sans fonctionnalité payante, et pré-remplir des bases pour guider la saisie manuelle.

## Quand l'utiliser
- On veut un dashboard **dans Notion** qui reflète une base en direct, plus riche que les graphiques natifs (ou sans plan payant).
- On veut que des bases (surtout à saisie manuelle) aient déjà **une ligne par entité**, titrée, prête à remplir.

## Patron A — Dashboard hébergé + page hybride
1. **Servir le HTML par un webhook** du moteur d'automatisation (n8n self-hosted à webhooks publics) : Webhook GET → lire la base (API HTTP) → Code qui **génère le HTML** depuis les données du moment → Respond `text/html`.
2. **Intégrer l'URL en bloc `<embed>`** dans la page cible. Mise à jour = à chaque chargement (données live), zéro snapshot.
3. **Page hybride (progressive enhancement)** car le conteneur cible (ex. embed Notion) peut **désactiver le JavaScript** :
   - Rendre le contenu **statique côté serveur** (cartes, courbes SVG avec valeurs écrites, table) → s'affiche même sans JS.
   - Ajouter un `<script>` qui, **si le JS tourne** (onglet navigateur), reconstruit une version **interactive** (clic, survol, toggles).
   - **Basculer statique→interactif seulement APRÈS un rendu réussi** + envelopper dans `try/catch` → jamais de page blanche.

### Difficultés
- Le conteneur d'embed sandboxe le JS → l'interactif n'existe qu'hors du conteneur.
- Générer du HTML dans un nœud Code = échappement pénible (guillemets) ; objets JSON littéraux interdits dans les expressions `={{ }}`.

### Solutions
- Une seule URL, deux comportements (statique dans l'embed, interactif au navigateur) + lien « ouvrir la version interactive ».
- Construire le HTML en concaténation, injecter les données via `JSON.stringify`, attributs HTML sans espaces non-quotés pour limiter l'échappement.

### Lessons
- « Interactif DANS l'outil » ≠ possible partout : vérifier la politique JS du conteneur AVANT de promettre.
- Un webhook qui régénère depuis la source = « hébergé + à jour » sans hébergeur tiers ni abonnement.
- Montrer le rendu dans l'outil cible tôt ; une maquette ne révèle pas les limites du conteneur.

## Patron B — Seeder de placeholders
1. Scanner les entités (ex. éditions ≠ annulées).
2. Pour chaque entité × chaque base cible, **find-by-title** ; si absent, **create** une ligne titrée `<entité> · <suffixe>` avec la relation posée et les mesures vides. Idempotent, jamais d'écrasement.
3. Aligner le titre sur celui qu'un runner auto utilise déjà → upsert commun (pas de doublon).

### Lesson transverse
- **Tout agrégateur en aval doit ignorer les placeholders vides** (ne compter que les valeurs non nulles) — sinon les lignes de seeding faussent les calculs (des « 0 » fictifs).
- Une **clé d'upsert dédiée et stable** (ex. « Edition key ») évite les collisions quand le titre d'affichage change (date, wording).

## Pièges Notion transverses
- `<embed>` = pas de JS. Graphiques natifs = payants.
- MCP ne sait pas archiver → `PATCH archived:true` par HTTP (mini-workflow jetable).
- Limite horaire de requêtes SQL `query_data_sources` (plan gratuit) → vérifier via l'API HTTP / les logs d'exécution.
- `create-view` (parent_page_id) crée une vue liée filtrée par API ; filtre relation par titre non fiable → filtrer par une clé texte.
