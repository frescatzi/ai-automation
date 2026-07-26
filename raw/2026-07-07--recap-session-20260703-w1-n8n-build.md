---
type: raw
title: "RECAP_Session_20260703_W1-n8n-Build"
source_url: "drive:17MYofoGOaOqV8yYeKL44Vz2-p3mQ36Z8"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Récap de session — 03.07.2026

**Sujet :** Construction du workflow n8n **AFTRSN-Event-Creator (W-1)** via automatisation navigateur (Claude in Chrome), à partir de `W1_Event-Creator_Flowchart_Spec_20260702.md`.
**Lien workflow :** https://n8n.aftersunpeople.com/workflow/kjuj3r0QVP6ZicIr
**Résultat :** les 14 nodes de la spec existent et sont connectés dans le canvas n8n. Le workflow n'est pas encore testé de bout en bout ni publié.

---

## Ce qui a été fait aujourd'hui

- Finalisation de l'arborescence Drive média (Photos/Videos × Raw/Edited) et reconnexion à la chaîne Notion.
- Construction du node Calendar (**Créer event**) : credential OAuth Google Calendar créée en réutilisant le client OAuth de Drive, calendrier `hello@aftersunpeople.com`, description avec lien du dossier Drive.
- Construction du node **Notion — Injecter checklist** : appel HTTP direct à l'API Notion (`POST /v1/blocks/{page_id}/children`) avec les 8 sections + 27 to-dos du template "EDITION TEMPLATE", tapés intégralement dans le corps JSON.
- Construction du node **Notion — Amorcer Budget & Finance** : un node Code (`Preparer lignes Budget`) génère 5 items (Venue, Artists, Production, Marketing, Media) avec l'id de la page Edition, puis un node Notion crée les 5 pages en relation.
- Ajout d'un node **Merge** pour synchroniser les deux branches parallèles (Calendar + Budget) avant la suite — sans ce node, les deux branches déclenchaient des exécutions séparées du node suivant au lieu d'une seule.
- Construction du node **Mémoire — Save Episode** : appel du sous-workflow `LUMINA-MEMORY-WRITE/WEBHOOK` (id `zu4jfZbmDz8trQLl`) via un node **Execute Workflow** (et non un HTTP Request malgré le nom de la spec — voir plus bas), avec le contrat exact `{brand, title, content, collection, knowledge_type, source, source_ref}`.
- Construction du node **Agréger résumé** (`{status, notion_url, drive_url, calendar_url}`).
- Construction du node **Erreur → réponse** sur la branche `false` du node `If`, avec `{status:"error", step, message}`.
- **Décision d'architecture :** les nodes 13/14 de la spec ("Set + Respond to Webhook" / "Respond to Webhook") ont été adaptés en simples nodes **Set**, car le trigger réel du workflow est `When Executed by Another Workflow` et non un Webhook — un node "Respond to Webhook" y échouerait à l'exécution. Le dernier node exécuté devient automatiquement la valeur de retour du sous-workflow.

## Challenges rencontrés et résolus

Voir le détail complet dans les deux marches à suivre (`AFTRSN_MarcheASuivre_Construction-n8n.md` et `Generique_MarcheASuivre_Construction-n8n-Navigateur.md`). En résumé :

1. Bug du `=` en trop dans les champs Expression (resourceLocator) — cassait les lookups Drive.
2. Champs de l'`Edit Fields` bloqués en mode Fixed avec du texte `{{...}}` non évalué.
3. Duplication de `}}` causée par l'auto-fermeture des accolades de l'éditeur CodeMirror.
4. Dossiers Drive `_` dupliqués créés involontairement pendant les tests (re-exécution en cascade depuis le trigger).
5. Mise en place du credential Google Calendar (guidage de l'utilisateur, jamais de saisie directe de secret).
6. Mauvais clics dans les menus déroulants du node HTTP Request (Authentication, dropdown "From list"/"By ID").
7. Panneau de recherche de node n8n qui s'affichait tronqué en bord d'écran à la première ouverture.
8. Perte de focus lors de la saisie des champs "value" juste après les champs "name" dans les nodes Set/Edit Fields.
9. Écart entre la spec ("Respond to Webhook") et le trigger réel du workflow (`executeWorkflowTrigger`) — résolu en vérifiant le workflow live plutôt que de suivre le doc à la lettre (invariant #5 de la passation).

## Ce qui reste à faire

- Exécuter et valider le node **Notion — Injecter checklist** de bout en bout (jamais testé pour éviter une nouvelle cascade de dossiers dupliqués).
- Lancer un test complet du workflow avec un vrai payload Maestro.
- Nettoyer manuellement les dossiers Drive `_` dupliqués issus des tests (plusieurs occurrences, non supprimées automatiquement par règle de sécurité).
- Publier ("Publish") le workflow une fois validé.
- Continuer la feuille de route : W-2 à W-7 (Venue-Coordinator, Artist-Coordinator, Marketing-Planner, Email-Responder, Budget-Tracker, Knowledge-Capture).

## Rappel deadline

**Venue + minimum 2 artistes confirmés avant le 18 juillet 2026** (J-21 par rapport à l'event du 08.08.2026). En dessous de cette date, la fenêtre promo passe de 3 à 2 semaines.

---
*Rédigé le 03.07.2026, en fin de session.*
