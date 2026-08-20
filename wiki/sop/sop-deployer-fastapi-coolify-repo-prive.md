---
type: wiki
title: "SOP générique — Déployer une app FastAPI sur Coolify depuis un repo GitHub privé"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-generique-deployer-app-fastapi-coolify-repo-github-prive-2026-08-17.md
related: ["SOP_installer-pgvector-sur-postgres-coolify", "sop-bridge-claude-code-mac-vps", "concept-n8n-credentials"]
updated: 2026-08-17
---

# SOP générique — Déployer une app FastAPI sur Coolify depuis un repo GitHub privé

**Portée :** générique, toute app Python/FastAPI servie par `uvicorn`, déployée sur Coolify depuis un dépôt GitHub privé. Aucun identifiant réel.

## Principe

Le code vit dans un repo Git privé (source de vérité). Coolify le télécharge au build, construit l'image depuis le `Dockerfile`, et sert l'app derrière un domaine HTTPS. Les secrets passent par les **variables d'environnement de Coolify**, jamais par le repo.

## Marche à suivre

1. Pousser le code sur un repo GitHub **privé** : `Dockerfile`, code applicatif, config. Exclure secrets, base locale, venv via `.gitignore`.
2. Dans Coolify, ajouter une source GitHub (OAuth ou token) qui voit le repo.
3. Créer une ressource « Private Repository » : choisir le repo + la branche, Build Pack = Dockerfile, port = celui de `uvicorn`.
4. Définir les variables d'environnement (dont les secrets) dans la ressource, référencées dans le compose par `${VAR}`.
5. Monter un volume persistant pour toute donnée qui doit survivre aux redéploiements.
6. Attacher le domaine (DNS pointe vers l'IP), laisser Coolify gérer le certificat.
7. Deploy, puis vérifier par l'URL publique (healthcheck 200, racine 401 si auth).

## Garde-fous

1. Ne jamais coller le `docker-compose` seul dans l'éditeur sans le code : le build context sera vide.
2. Ne jamais exposer le port directement ; passer par le domaine HTTPS du proxy.
3. Ne jamais versionner de secret (clés API, mots de passe) dans le repo, même privé.
4. Toujours avoir un middleware d'auth avant d'exposer publiquement.

## Difficultés rencontrées & solutions

| Difficulté | Solution |
|---|---|
| « no available server » / 404 : DNS OK mais aucun conteneur déployé derrière | Passer par le repo Git (pas de collage manuel du compose) |
| Build bloqué « Helper container not yet started » (souvent disque plein) | Nettoyer les volumes/images orphelins, redémarrer Coolify, relancer le build |
| Conteneur « Exited » sans logs (échec de build, code absent du contexte) | Vérifier par l'URL publique, pas par le statut affiché |

## Leçons clés

- Le build context doit contenir le code : le repo Git est la source de vérité.
- Libérer le disque avant tout build Docker.
- Le test de vérité est l'URL publique (200/401), pas l'interface Coolify.

## Voir aussi

- [[SOP_installer-pgvector-sur-postgres-coolify]] — autre patron de déploiement/administration sur la même plateforme Coolify (base Postgres plutôt qu'app FastAPI).
- [[sop-bridge-claude-code-mac-vps]] — exemple d'app servie derrière un domaine HTTPS (Operator OS/FastAPI) qui consomme ce patron de déploiement.
- [[concept-n8n-credentials]] — même principe de gestion des secrets (jamais dans le repo, toujours en variable d'environnement/credential) appliqué côté n8n.
