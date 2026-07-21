---
type: raw
title: "POS-GENERIQUE_runner-releve-metriques-plateforme-vers-base-metrics_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-releve-metriques-plateforme-vers-base-metrics_2026-07-19.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Runner de relève de métriques : plateforme externe → base Metrics par canal

**Date :** 19.07.2026 · **Contexte d'origine :** AFTRSN P3 · M5b Newsletter Metrics · **Type :** POS-GÉNÉRIQUE (réutilisable).

> Patron : un contenu est publié à la main sur une plateforme (emailing, site, réseau) ; un runner planifié va chercher les chiffres via l'API de la plateforme et les dépose dans une base « Metrics » dédiée au canal, reliée à la ligne de contenu et à l'entité.

---

## 1. Difficultés
1. **Relier l'objet plateforme (campagne, page, post) à la ligne de contenu Notion** : l'objet est créé à la main, aucun ID n'existe côté Notion au départ.
2. **Quand relever, et jusqu'à quand ?** Des relèves infinies gaspillent des appels ; une relève unique fige trop tôt.
3. **Éviter les doublons de lignes Metrics** au fil des relèves quotidiennes.
4. **Tester un trigger planifié** : non déclenchable par API n8n.
5. Les stats d'un objet **pas encore actif** (campagne non envoyée) reviennent partielles → risque de plantage ou de fausses valeurs.

## 2. Solutions (le patron)
1. **Matching à deux étages** : (a) champ `ID` sur la ligne de contenu, prérempli à la main OU réécrit par le runner après premier match = source de vérité ; (b) sinon **convention de nommage** — l'objet plateforme est nommé en commençant par le code de l'entité (ex. code édition) → le runner liste les objets, matche par préfixe, départage par proximité de date, puis **écrit l'ID sur la ligne** (le matching ne se refait plus).
2. **Fenêtre de relève par DATE, pas par mutation** : filtrer `Status=Sent/Published` ET `date ≥ J-N`. La ligne sort de la fenêtre toute seule à J+N — zéro état à gérer, relève quotidienne entre-temps (les chiffres mûrissent).
3. **Upsert par TITRE** : la ligne Metrics porte le même titre que la ligne Content ; find-par-titre → update sinon create. 1 objet = 1 ligne, mise à jour datée (`As-of`).
4. **Relations posées à la création** (Content + entité) : la navigation « entité → contenu → chiffres » marche dans les deux sens.
5. **Écritures Notion en HTTP brut** quand les propriétés sont number/date (le nœud Notion n8n est sûr pour select/rich_text/relation/title ; au-delà, POST/PATCH `/v1/pages` avec un JSON construit en nœud Code).
6. **Tests d'un runner planifié** : basculer le schedule en 1 min le temps du test, puis remettre la vraie cadence. Séquence de preuve : croisière (0 éligible → 0 action) → create → **update au tick suivant** (anti-doublon) → données réelles.
7. **Vérifier la permission API avant le build** : un GET de lecture jetable (workflow webhook temporaire avec la credential cible) tranche en 30 secondes.
8. Défauts à 0 pour les blocs de stats absents (objet pas encore actif) — écrire des zéros datés vaut mieux que planter.

## 3. Lessons
- **La convention de nommage est un contrat humain** : elle doit être triviale (« commence par le code édition ») et rattrapable (champ ID prérempli = prioritaire). Le runner réécrit l'ID dès le premier match pour ne plus jamais dépendre du nom.
- **Le drainage par date est le plus simple des drainages** : aucune mutation, aucun compteur — la fenêtre glissante fait tout.
- **Content et Metrics séparés mais jumelés par le titre** : la séparation des environnements (décision produit) n'empêche pas un appariement trivial si les titres sont canon.
- Les IDs utiles (campagne, lien analytics) méritent d'être **réécrits sur la ligne Content** : l'humain y gagne le lien cliquable, le runner y gagne l'idempotence du matching.
- Un test « fini » peut être **mal attribué** : vérifier avec l'humain À QUELLE entité appartient l'objet réel (ici la campagne était celle d'EP05, pas d'EP06) avant de laisser l'état en production.
