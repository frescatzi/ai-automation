---
type: wiki
title: "SOP générique — Synchroniser une base SQLite entre deux conteneurs Docker (docker cp + cron)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-generique-synchroniser-sqlite-entre-deux-conteneurs-docker-2026-08-17.md
updated: 2026-08-17
related: ["sop-dream-engine-cron-prescriptions-llm", "concept-classification-workflows-n8n"]
---

# SOP générique — Synchroniser une base SQLite entre deux conteneurs Docker (docker cp + cron)

**Idée centrale :** patron réutilisable, hors marque, pour tout cas où un conteneur **producteur** génère une base SQLite qu'un conteneur **serveur** distinct doit exposer, sans volume partagé entre les deux. On synchronise par copie planifiée (`docker cp` via un fichier temporaire hôte + cron) plutôt que par volume partagé, plus lourd à mettre en place pour un besoin non temps-réel. Implémentation concrète connue : [[sop-dream-engine-cron-prescriptions-llm]] (Operator OS / Dream Engine, §3).

## Marche à suivre

1. Identifier les deux conteneurs (`docker ps`) et leurs chemins internes respectifs.
2. Écrire un script **hôte** :
   ```
   docker cp producteur:/chemin/base /tmp/base
   docker cp /tmp/base consommateur:/data/base
   docker restart consommateur
   ```
3. Trouver le conteneur consommateur **dynamiquement** (grep sur un préfixe stable de son nom), car ce nom change à chaque redéploiement.
4. Rendre le script exécutable, l'ajouter au crontab avec une marge après le job producteur.

## Garde-fous

1. Docker ne supporte **pas** la copie directe entre deux conteneurs (`docker cp A:… B:…` échoue) : toujours passer par un fichier intermédiaire sur l'hôte (`/tmp`).
2. Le chemin interne du producteur (ex. `/opt/data`) n'existe pas sur l'hôte : ne jamais tenter un chemin hôte direct, toujours passer par `docker cp`.
3. Ajouter la ligne crontab **sans** `crontab -e` interactif (source d'erreur si l'éditeur est mal quitté) : utiliser `( crontab -l; echo "..." ) | crontab -`.

## Difficultés rencontrées & solutions

| Erreur observée | Cause | Solution |
|---|---|---|
| `lstat /opt/data: no such file or directory` | chemin interne au conteneur confondu avec un chemin hôte | passer systématiquement par `docker cp` |
| `copying between containers is not supported` | `docker cp` direct conteneur→conteneur refusé par Docker | copie en deux temps via `/tmp` sur l'hôte |
| `crontab -e` n'enregistre pas la ligne | éditeur interactif mal quitté | écrire la ligne par redirection/pipe (`crontab -l \| ... \| crontab -`) |

## Leçons clés

- Un volume partagé est une solution plus propre mais plus lourde à mettre en place ; une synchro **cron quotidienne** (ou à l'intervalle voulu) suffit pour un besoin non temps-réel.
- Toujours **tester le script à la main** avant de le cronner.

## Voir aussi

- [[sop-dream-engine-cron-prescriptions-llm]] — implémentation concrète de ce patron : synchro `operator.db` du conteneur Hermes vers le conteneur consommateur, cron hôte à 7h15.
- [[concept-classification-workflows-n8n]] — ce type de cron hôte de synchro se classe `no_agent` (script planifié, pas un appel d'agent) dans la nomenclature Lumina.
