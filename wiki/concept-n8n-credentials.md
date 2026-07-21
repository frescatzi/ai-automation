---
type: wiki
title: Credentials n8n — gestion et bonnes pratiques
status: active
publish: notion
vault: ai-automation
brand:
sources: []
related:
  - wiki/synthese-oauth2-n8n-google.md
  - wiki/concept-oauth2-automation.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
  - wiki/sop/n8n-Brancher-API-et-Premier-Workflow.md
  - wiki/sop/sop-reparer-credential-postgres-partagee-n8n.md
  - wiki/sop/sop-token-systemuser-meta-ads-n8n.md
  - wiki/sop/sop-repondeur-email-drafts-agent.md
updated: 2026-07-21
---

# Credentials n8n — gestion et bonnes pratiques

> Page stub — à enrichir à partir d'une source raw/ dédiée.

## Idée centrale

n8n centralise tous les accès aux services externes dans des **credentials** réutilisables : clés API (Anthropic, OpenAI, Gemini) et tokens OAuth2 (Google Drive, Gmail…). Un credential bien configuré peut être partagé entre plusieurs workflows sans être ré-saisi.

## Types de credentials courants

- **API Key** : clé statique (ex. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`). Simple, risquée si exposée.
- **OAuth2** : flux Authorization Code. n8n agit comme client OAuth — voir [[concept-oauth2-automation]] et [[synthese-oauth2-n8n-google]] pour la procédure Google.
- **HTTP Header Auth / Basic Auth** : pour APIs sans SDK dédié.
- **Token System User longue durée** (ex. Meta Ads) : ni clé API statique ni flux OAuth2 Authorization Code — un token généré côté fournisseur pour un « utilisateur système », expiration « Jamais », collé directement dans un credential natif. Voir [[sop/sop-token-systemuser-meta-ads-n8n]].

## Bonnes pratiques

- Ne jamais coller une clé API dans un nœud en dur — toujours utiliser un credential nommé.
- En équipe, préférer les **variables d'environnement n8n** (`N8N_ENCRYPTION_KEY`, etc.) pour éviter que les secrets soient stockés en clair dans la DB SQLite.
- Révoquer et régénérer les clés compromises immédiatement depuis la console du fournisseur.
- Pour les credentials OAuth2 Google en mode Test : anticiper l'**expiration des refresh tokens à 7 jours** (voir [[concept-oauth2-automation]]).

## À documenter (trou actuel)

- Procédure de rotation d'une clé API Anthropic dans n8n sans interrompre les workflows actifs.
- Export/import de credentials entre instances n8n.

## Panne : credential Postgres partagée fantôme

Une credential Postgres unique câblée dans de nombreux workflows (mémoire + Chat Memory de tous les agents) est un point unique de défaillance : si elle se corrompt (« credential fantôme » — listée mais 404 au chargement), tout le système mémoire/agents tombe. Procédure complète de diagnostic, repoint en masse par API et republication : [[sop/sop-reparer-credential-postgres-partagee-n8n]].

## Piège : `403 Forbidden` malgré une credential valide

Un `403 Forbidden` sur une API externe (ex. API email) alors que la credential n8n est correctement configurée n'est pas forcément un problème de credential — c'est souvent un **prérequis d'infrastructure manquant côté fournisseur** (API non activée dans sa console cloud). Vérifier l'activation de l'API avant de déboguer le credential ou le workflow. Cas rencontré : [[sop/sop-repondeur-email-drafts-agent]].

## Voir aussi

- [[sop/Guide-Connexion-Agents-AI-n8n]] — créer les credentials Anthropic / OpenAI / Gemini pas à pas.
- [[sop/n8n-Brancher-API-et-Premier-Workflow]] — brancher et tester le premier workflow avec ces credentials.
- [[concept-oauth2-automation]] — patron OAuth2 générique utilisé par les credentials de type Google.
- [[synthese-oauth2-n8n-google]] — guide spécifique Google Drive / Gmail dans n8n.
- [[sop/sop-reparer-credential-postgres-partagee-n8n]] — runbook de réparation d'une credential Postgres partagée fantôme.
- [[sop/sop-token-systemuser-meta-ads-n8n]] — token System User Meta Ads : procédure, pièges, credential natif « Facebook Graph API ».
- [[sop/sop-repondeur-email-drafts-agent]] — piège API désactivée côté fournisseur mimant une credential invalide.
