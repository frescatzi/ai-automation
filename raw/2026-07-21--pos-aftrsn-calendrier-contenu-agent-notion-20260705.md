---
type: raw
title: "POS-AFTRSN_Calendrier-Contenu-Agent-Notion_20260705"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_calendrier-contenu-agent-notion_20260705.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-AFTRSN — Générateur de calendrier de contenu (agent → Notion, draft-only)

**Date :** 2026-07-05 (snapshot d'un état vérifié en production)
**Portée :** AFTRSN · processus P2-support / W-4 Marketing-Planner (cœur v1).
**Objet :** produire, à la demande et par édition, un calendrier de contenu Instagram ~3 semaines avec copies prêtes à publier + un draft de newsletter, entièrement en **brouillon** dans Notion, sans jamais rien publier automatiquement.

Voir la version débarrassée des éléments AFTRSN : `POS-GENERIQUE_Calendrier-Contenu-Agent-Base_20260705.md`.

---

## 1 · Architecture (n8n)
`Trigger {edition}` → `HTTP Lire édition` (Notion query, titre = edition) → `Code Brief & calendrier` (calcule les créneaux relatifs à la date d'édition + assemble le prompt de rédaction) → `IF Édition trouvée ?` → `HTTP Vérifier existant` (idempotence par relation) → `Code Décider` → `IF Déjà généré ?` → `ExecuteWorkflow Agent-Marketing (query)` → `Code Parser` (JSON strict → 1 item par post + 1 newsletter, chaque item porte son `notionPageBody` complet) → `HTTP Créer page` (POST /v1/pages, une fois par item) → `Code Compter` → `Telegram` → `ExecuteWorkflow Mémoire` → `Set Résumé`.

Deux branches terminales : `Erreur - introuvable` (édition inconnue) et `Telegram - Deja fait` (calendrier déjà présent).

## 2 · Points clés
- **Déclenchement humain, par édition** — le lancement de campagne est une décision, pas un automatisme.
- **API Notion brute (HTTP Request)** en lecture ET écriture (`predefinedCredentialType: notionApi`, header `Notion-Version`), pour maîtriser relation/date/blocs et éviter les clés simplifiées `property_*`.
- **Le Code construit le payload complet** de chaque page (`parent`, `properties`, `children` blocs) ; le node HTTP ne fait que `POST JSON.stringify($json.notionPageBody)`, une fois par item. Élégant et déterministe.
- **Copie dans le corps de page** (blocs paragraphe) car la base n'a pas de champ texte long ; une propriété `Publish date` (date) est ajoutée pour porter le calendrier.
- **Idempotence par relation** : requête de la base contenu filtrée sur la relation vers la page édition ; si non vide → on ne recrée pas.
- **Draft-only** : `Status = Draft` partout ; validation et publication restent humaines.

## 3 · Difficultés rencontrées
1. `jsonBody = {{ JSON.stringify({ ...objet littéral... }) }}` → erreur n8n « invalid syntax ».
2. Base contenu sans champ date ni champ copie.
3. Réponse d'agent parfois entourée de texte / fences.
4. Bases Notion en doublon (même nom, IDs différents).

## 4 · Solutions implémentées
1. JSON **littéral** avec interpolation dans les valeurs : `={"filter":{"property":"Edition","title":{"equals":"{{ $json.edition }}"}}}`.
2. Ajout d'un champ `Publish date` (PATCH DB) ; copie écrite dans le corps de la page.
3. Parseur robuste : retrait des fences, isolation `{`…`}`, erreur explicite si non parsable.
4. Sélection **By ID** systématique ; l'ID canonique = celui utilisé par les workflows en prod (1re occurrence de la recherche).

## 5 · Leçons apprises
- Ne jamais mettre un **objet littéral** dans une expression de jsonBody n8n ; préférer le JSON en clair avec `{{ champ }}` dans les valeurs.
- **Lire le schéma réel** de la base cible avant d'écrire ; ajouter explicitement les champs manquants dont le process a besoin (ici, une date).
- Faire **construire le payload d'écriture par un node Code**, le node HTTP restant « bête » : plus lisible, plus testable, réutilisable pour tout POST par item.
- L'**idempotence par relation** est le garde-fou naturel d'un générateur relancé à la demande.
- Pour pousser du **code non trivial par API**, encoder en base64 (décodage UTF-8) évite tous les pièges d'échappement.
