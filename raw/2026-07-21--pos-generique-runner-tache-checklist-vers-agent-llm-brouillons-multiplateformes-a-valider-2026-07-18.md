---
type: raw
title: "POS-GENERIQUE_runner-tache-checklist-vers-agent-LLM-brouillons-multiplateformes-a-valider_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-tache-checklist-vers-agent-llm-brouillons-multiplateformes-a-valider_2026-07-18.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-GÉNÉRIQUE · Runner : d'une tâche de checklist → agent LLM → brouillons multi-plateformes à valider

**Date :** 18.07.2026 · **Contexte d'origine :** AFTRSN Phase 3 M2. **But :** patron réutilisable pour faire exécuter une tâche de contenu par un agent LLM et déposer des brouillons standardisés dans une base « à valider », sans jamais publier.

## Quand l'utiliser
Un automate déterministe (watcher) pose des tâches dans une base. On veut qu'un **agent LLM** prenne une tâche donnée, produise plusieurs livrables de contenu (un par canal), et les dépose en `À valider` — chaque canal présentant la **même idée** différemment. Aucune sortie externe.

## Principe : séparer planification et exécution
- Le **watcher** reste déterministe (pas d'appel LLM dedans) et se contente de poser les tâches.
- Un **runner séparé** écoute la base de tâches, filtre sur *son* type de tâche, appelle l'agent, écrit les brouillons. Chaque type de livrable = un runner → fan-out propre et découplé.

## Étapes
1. **Déclencheur** : Notion Trigger(s) sur la base de tâches (Added + Updated, poll). Le trigger voit toute la page → relire la tâche puis **filtrer** (type de tâche via le titre, Status = à-faire, exécutant = agent, relation vers l'entité parente présente ; sinon renvoyer `[]` pour ne rien faire).
2. **Recontextualiser** : relire l'entité parente (l'édition/le projet) via la relation, et **reconstruire le brief en code** (ne pas dépendre d'un brief persisté).
3. **Standard de contenu** : imposer à l'agent (a) une **idée maîtresse unique** (angle + émotion) décidée en premier, (b) sa **déclinaison** sur chaque champ plateforme (mêmes infos/ressenti, forme adaptée), (c) les **contraintes par champ** (longueurs, ton, règles de marque), (d) une **sortie JSON stricte**.
4. **Appel agent** : Execute Sub-workflow vers l'agent — **vérifier le nom exact du champ d'entrée** de son trigger (ex. `query`), ne pas le supposer.
5. **Garde-fou** : parser le JSON (tolérant aux fences), valider la présence/non-vacuité de toutes les clés. Non conforme → **marquer la tâche `Bloqué`** + raison, **rien** d'écrit (jamais de contenu partiel).
6. **Écriture** : créer la page dans la base de contenu avec **un champ par livrable** (statut `À valider`), + un corps de page lisible.
7. **Porte + idempotence** : faire passer la **tâche** de `à-faire` → `à valider` (sert de signal humain ET de garde anti-doublon : le runner ne reprend que les `à-faire`). **Jamais** de statut terminal écrit par l'automate.

## Difficultés (génériques)
- Le contrat d'entrée réel d'un sous-workflow diffère souvent de la doc.
- La base de contenu peut manquer du statut « à valider » et de champs texte adéquats.
- Sans consigne, les textes de canaux différents divergent.
- Les triggers de poll ne sont pas déclenchables par API ; leur « fetch test event » est non déterministe.
- Certains types de blocs (divider…) ne sont pas couverts par le node natif.
- Un multi_select n'accepte pas une valeur inconnue par l'API page.

## Solutions (génériques)
- Lire le sous-workflow pour le vrai nom de champ d'entrée.
- Étendre le schéma de façon **additive** (statut + champs), en s'appuyant sur une **doc de référence** pour le standard.
- Champ « idée maîtresse » explicite + consigne en 2 temps (fixer puis décliner).
- **Harness Manual Trigger jetable** injectant l'ID cible pour un test déterministe.
- **API native par HTTP** (auth par credential prédéfini) pour les blocs non couverts, en **non bloquant**.
- Ajouter l'option au schéma du multi_select avant d'écrire la valeur.

## Lessons (génériques)
- Vérifier les contrats d'intégration avant de câbler ; ne pas se fier aux passations.
- **Standardiser le contenu par plateforme avant de générer** ; une idée maîtresse commune fait la cohérence de campagne.
- La **porte de validation peut porter l'idempotence** (pas de champ caché nécessaire).
- Un agent LLM doit rendre du **JSON strict** derrière un **garde-fou** ; jamais de sortie partielle publiée.
- Garder les automates **sans « terminé »** : la clôture reste humaine.
