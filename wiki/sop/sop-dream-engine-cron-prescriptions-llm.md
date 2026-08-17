---
type: wiki
title: "SOP — Dream Engine : cron quotidien de prescriptions LLM (sévérité × impact $)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-exact-operator-os-dream-engine-cron-prescriptions-2026-08-17.md
updated: 2026-08-17
related: ["concept-classification-workflows-n8n", "concept-hermes-agent", "synthese-lumina-ai-os", "sop-synchroniser-sqlite-deux-conteneurs-docker", "sop-bridge-claude-code-mac-vps"]
---

# SOP — Dream Engine : cron quotidien de prescriptions LLM (sévérité × impact $)

**Idée centrale :** pattern réutilisable pour transformer des données opérationnelles brutes en un **brief actionnable quotidien** : un cron re-collecte les données, interroge un LLM avec un contrat de sortie strict, et stocke un nombre fixe de « prescriptions » classées par sévérité et impact financier. Implémentation de référence : **Dream Engine** d'Operator OS/LUMINA — ✅ verrouillé, validé live au 2026-08-17.

## 1. Flux quotidien

1. Cron hôte **`Operator OS Dream Engine`** (7h00, classé `no_agent` — script planifié, pas un appel d'agent) → lance `operator-dream.sh`.
2. Étapes de re-collecte : `schema` → `collect` → `skills` → `derive_usage` → `memory_ingest`.
3. `collector/dream.py` interroge Claude via `/query-claude` → parse la réponse JSON → insère les lignes dans la table `dream_prescriptions`.
4. Sortie livrée : **4 prescriptions**, chacune avec icône de sévérité, message, action et impact `$`/mois.

## 2. Contrat de sortie LLM (format de la prescription)

- JSON par prescription : `{dimension, severity(high|medium|low), dollar_impact, title, message, action}`.
- Dimensions autorisées (liste fermée) : `conversation, cost, skills, memory, session, workflow, external, business`.
- Contraintes imposées par le prompt (pas par schéma structuré/tool-call) : **exactement 4 objets**, pas de markdown, pas de tiret cadratin.

## 3. Synchronisation cross-conteneur

- Le Dream Engine écrit dans `/opt/data/operator-os/operator.db`, à l'intérieur du conteneur **Hermes**.
- Un cron **hôte** séparé, 7h15, `sync-operator.sh` : `docker cp` Hermes → `/tmp` → conteneur cible, puis **restart** du conteneur cible pour qu'il recharge la base fraîche.
- Pattern générique : quand une donnée est produite dans un conteneur et consommée par un autre **sans volume partagé**, la synchro `docker cp` + restart planifiés est une solution simple et suffisante pour un besoin non temps-réel. Marche à suivre détaillée, garde-fous et pièges : [[sop-synchroniser-sqlite-deux-conteneurs-docker]].

## 4. Pièges rencontrés & correctifs

- **Piège — tri inversé :** un `ORDER BY id DESC` inversait l'ordre de sévérité que Claude avait pourtant produit dans le bon ordre.
  **Correctif :** ne lire que le dernier run (`WHERE run_at = max(run_at)`) et `ORDER BY id ASC` pour préserver le classement d'origine du LLM.
- **Piège — base au mauvais endroit :** la base vit dans le conteneur Hermes, pas dans le conteneur qui la consomme → sans synchro hôte dédiée, le consommateur ne voit jamais les nouvelles prescriptions (cf. §3).

## 5. Leçons clés

- Le Dream Engine est un **brief périodique, pas un tableau de bord temps réel** : une synchro quotidienne suffit, inutile de sur-ingénierer un temps réel.
- **Claude refuse de conclure sur des données trop pauvres** (ex. observé sur une entité peu alimentée) — c'est le comportement voulu : un garde-fou contre les prescriptions hallucinées sur données insuffisantes, pas un bug à corriger.

## Voir aussi

- [[concept/concept-classification-workflows-n8n]] — classification « planifié / no_agent » de ce type de cron dans la nomenclature Lumina.
- [[concept/concept-hermes-agent]] — le conteneur/hôte Hermes mentionné en §3 est l'infrastructure qui héberge aussi Hermes-Agent ; à ne pas confondre : ce cron s'exécute en dehors du chemin d'exécution de l'agent (`no_agent`).
- [[synthese/synthese-lumina-ai-os]] — place d'Operator OS/Dream Engine dans l'écosystème LUMINA plus large.
- [[sop/sop-bridge-claude-code-mac-vps]] — autre composant Operator OS interrogeant Claude, sur la même infra VPS.
