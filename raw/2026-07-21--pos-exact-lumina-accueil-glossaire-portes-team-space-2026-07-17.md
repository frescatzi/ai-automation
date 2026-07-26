---
type: raw
title: "POS-EXACT_LUMINA-accueil-glossaire-portes-team-space_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_lumina-accueil-glossaire-portes-team-space_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT : LUMINA, page d'accueil + glossaire + portes filtrées, et renommages Team Member / Team Space

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) / LUMINA AI OS · Espace : Notion (teamspace LUMINA AI OS - HQ + The Backstage)
Rôle : marche à suivre exacte de la session « donner à LUMINA une vraie page d'accueil et un glossaire, et figer la terminologie Team Member / Team Space ». Validé par Karter le 17.07.2026.

## 1. Contexte

Suite du déménagement LUMINA. Objectifs : (1) renommer « Human » en « Team Member » (plus propre, plus empathique) sur le champ Executor et dans la doc ; (2) donner au teamspace LUMINA une page d'accueil qui explique ce qu'est LUMINA et la nomenclature des documents, plus un glossaire des termes techniques ; (3) sortir la base de connaissances de l'ancienne coquille « The Backstage (not in use) ».

## 2. Étapes exactes

1. Executor « 🧑 Human » devient « 🧑 Team Member » (Karter, UI Notion). Le renommage d'option préserve l'identité de l'option : les tâches existantes gardent leur valeur, les filtres de vue ne cassent pas. Base Tasks, data source `e50f931d-3f8d-483f-895a-c09408455128`.
2. Défaut de template. Template « New Task » (`39e24f2fef0c802695c8ccb3accb22e0`) : Executor par défaut « 🧑 Team Member » (API `update-page`, `update_properties`). Alternative gratuite à l'automation Notion (payante, écartée). Le côté « 🤖 Agent » sera posé par les workflows n8n qui créeront des tâches.
3. Page « Human Space » devient « Team Space » (`39124f2fef0c811599edec1c18b44b20`, API `update-page` titre). Textes nettoyés : intro de la page et description « Zones » de l'accueil Backstage (`39124f2fef0c8172a961c8a57721ff0f`) ; tirets longs retirés.
4. Base sortie de la coquille : « AI Automation Knowledge » (`6fac8cb8-91e4-4c74-9e37-a86419826905`, ds `657827e3-1e2a-48d5-8f52-5954e3618687`) draguée par Karter de sous « The Backstage (not in use) » (`38124f2fef0c81aabb40c304b4e438c5`) vers la racine du teamspace LUMINA AI OS - HQ (`38224f2f-ef0c-81f9-9080-0042b67d3909`). Vérifié : ancestor-path vide, IDs base et ds inchangés, schéma et lignes intacts.
5. Page d'accueil « 🌙 LUMINA AI OS » (`3a024f2f-ef0c-818e-a7ba-e94eed811bd0`) créée par API, bilingue EN/FR : définition de LUMINA (plateforme d'automatisation multi-marques ; AFTRSN n'est qu'une des marques ; roster AFTRSN, Karter Amaris, KOWRIS, Lumina Agency, environnement personnel, banque de connaissances), lien vers la base, nomenclature des documents (SPEC-BUILD, POS-EXACT, POS-GÉNÉRIQUE, PASSATION, CONCEPT, CARTOGRAPHIE, BIBLE 360°) avec POS = Procédure Opérationnelle Standardisée (la SOP en français), et lien vers le glossaire. Créée provisoirement sous la coquille, puis draguée à la racine par Karter.
6. Glossaire « 📖 Glossary / Glossaire » (`3a024f2f-ef0c-818a-be3a-dcde63d92cfe`) créé en sous-page de l'accueil, bilingue, une trentaine de termes (SOP, POS, MVP, LLM, n8n, MCP, RAG, pgvector, embedding, OAuth2, idempotent, webhook, Gateway, Router, Maestro, Hermès, Lyra, Executor, teamspace, workspace, data source, credential, token, Vault, Coolify).
7. Portes filtrées : deux vues board sur la base, « By Vault » (`view://3a024f2f-ef0c-8146-811f-000cfbd14697`) et « By Type » (`view://3a024f2f-ef0c-8110-9b26-000ce38e215b`), posées en ONGLETS sur la base (pas incrustées sur l'accueil, pour garder la sidebar propre).
8. Base renommée « AI Automation Knowledge » (tiret long retiré, API `update-data-source` titre).
9. Coquille « not in use » mise à la corbeille par Karter (base et accueil déjà sortis ; ses enfants legacy Brand Source of Truth et Archive partis en corbeille avec elle, récupérables).

## 3. Invariants tenus

1. Additif avant destructif : contenu créé et base sortie AVANT toute suppression ; suppression = geste Karter, en dernier.
2. Secrets jamais touchés ; credential Backstage `AFTRSN-n8n-Backstage` non concerné.
3. Renommages ID-stables : aucun workflow n8n recâblé.

## Difficultés rencontrées

1. L'API Notion ne peut ni créer ni déplacer à la RACINE d'un teamspace (parent non adressable) : ni la base, ni la page d'accueil.
2. Renommer une option de select par API réécrit toute la liste d'options (risque de vider le champ sur les lignes existantes).
3. Le remplissage automatique conditionnel (automation Notion) est une fonctionnalité payante, indisponible sur le plan.
4. La doc antérieure disait « supprimer The Backstage (not in use) » alors que la base vivait encore dedans (piège de suppression).

## Solutions implémentées

1. Création sous une page adressable (la coquille) puis drag propriétaire à la racine ; idem pour la base. Le geste humain de 2 secondes bat la lutte contre l'API.
2. Renommage d'option fait dans l'UI par Karter (préserve l'identité de l'option et les valeurs des lignes).
3. Défaut de template « New Task » = Team Member (gratuit) en remplacement de l'automation payante ; le côté Agent est délégué à n8n.
4. Sortir la base d'abord (drag) : la coquille devient supprimable sans danger ; suppression laissée à Karter.

## Lessons learned

1. La racine d'un teamspace n'est pas adressable par l'API : créer sous une page existante, puis draguer à la racine.
2. Renommer une option de select : préférer l'UI (préserve l'identité) à l'API (réécrit la liste).
3. Le ciblage par ID survit aux déplacements et aux renommages ; un titre de base peut être nettoyé sans impact n8n.
4. Des onglets de vues sur la base valent mieux que des vues incrustées sur une page : pas d'encombrement de la sidebar (loi sidebar Notion).
5. Documenter la vérité terrain : la base était sous « not in use », pas à la racine comme le disait la doc ; toujours re-fetch avant d'agir.
