---
type: raw
title: "MILESTONE_Clarification-Hermes-et-processus-connaissance_2026-07-03"
source_url: "drive:11wUcMrMzkC-NSBRyoVt-YnotIZmh5P2g"
captured: 2026-07-06
vault: brands
brand: AFTRSN
immutable: true
---

# MILESTONE — Clarification Hermès + correctif du processus de connaissance (2026-07-03)

**Statut : résolu au niveau documentation ; effectif côté agents seulement après ingestion du SYSTEM-CANON dans `canon` (voir §4).**

## 1. Ce qui s'est passé

Pendant la discussion « monitoring publicitaire », le nom « Hermès » a déclenché trois hypothèses successives dans un autre chat : (1) agent du roster — faux ; (2) sous-outil privé de Marketing (réponse de Maestro) — incomplet ; (3) plateforme d'exécution séparée — exact. L'investigation (Notion, interrogation de Maestro, lecture du workflow `LUMINA-Hermes-Exec`, visite de `hermes.aftersunpeople.com`) a confirmé l'hypothèse 3.

## 2. Corrections factuelles apportées au compte-rendu du 03-07

Le compte-rendu de l'autre chat contenait trois erreurs, corrigées ici :

1. **Le roster compte 7 spécialistes, pas 6** : il manquait **AFTRSN-CHANNEL-CONTENT** (contenu social + newsletter, publié, câblé sur Maestro — vérifié node par node le 02-07, cf. Bible 360°). La page Notion « 🤖 Agents » mise à jour avec « Maestro + 6 » doit être re-corrigée en « Maestro + 7 ».
2. **Hermès n'est pas appelé que par Marketing** : **AFTRSN-Secretary** possède aussi l'outil `Call 'Hermes (Ops)'`. Décision Karter du 03-07 : Hermès = ressource commune, à terme accessible par tous les agents (même catégorie qu'un LLM externe type ChatGPT/Claude/Gemini).
3. **« Sauvegardé en mémoire épisodique » ≠ corrigé** : `memory-search` exclut `episodic` par défaut, et la Chat Memory est scopée par session. La correction relayée à Maestro/Marketing ne survivra pas à une nouvelle session tant qu'elle n'est pas dans `canon`.

## 3. Le vrai problème de process (et sa règle corrective)

**Problème** : les agents n'avaient aucune connaissance canonique de leur propre système (roster, Hermès, architecture). Chaque interrogation produisait donc une improvisation différente, et les « corrections » circulaient par épisodes/chats — un canal qui ne persiste pas. Risque d'amplification : chaque nouvelle session repart de la version fausse et la re-documente ailleurs (ex. Notion).

**Règle actée (03-07-2026)** :
- Épisode = journal. **Jamais** un canal de correction de la connaissance.
- Toute vérité durable sur le système = mise à jour du **SYSTEM-CANON_LUMINA** + réingestion en `collection='canon'` dans les banques (`aftrsn_memory` + `lumina_memory`).
- Hiérarchie des sources : n8n live > SYSTEM-CANON/Bible > réponses d'agents > épisodes.
- Un agent qui ne trouve pas l'info dans le canon dit « je ne sais pas » au lieu de supposer.

## 4. Actions

- [x] SYSTEM-CANON_LUMINA_2026-07-03.md rédigé (roster à 7, Hermès ressource commune, gouvernance skills, règle de connaissance) — double-sauvé projet Claude + Drive.
- [ ] Ingestion du SYSTEM-CANON en `canon` dans `aftrsn_memory` et `lumina_memory` (via LUMINA-MEMORY-INGEST-TEXT).
- [ ] Re-corriger la page Notion « 🤖 Agents » : Maestro + **7** spécialistes.
- [ ] Vérifier le compte exact de workflows (32 relevés le 02-07, « 33 » aperçus le 03-07) et mettre la Bible à jour si un workflow est apparu.
- [ ] (Décision Karter à exécuter) Câbler `Call 'Hermes (Ops)'` sur les 5 agents qui ne l'ont pas encore (Culture, Experience, Channel-Content, Comptable, Qualité) — modification de workflows PROD, à faire avec validation.
- [ ] Chercher/créer le skill « monitoring performance publicitaire + optimisation » dans la bibliothèque Hermès (gouvernance : proposition + validation avant toute action critique).

## 5. Renvois

- SYSTEM-CANON_LUMINA_2026-07-03.md (la version agents)
- BIBLE_LUMINA-OS_360_2026-07-02.md (la version humaine détaillée)
- POS-AFTRSN_Skills-library-Hermes-apprenant_2026-07-02 · POS-AFTRSN_Hermes-outil-Ops-agents_2026-07-02

*MILESTONE — 2026-07-03, Claude (Cowork), avec vérifications live n8n du 02-07.*
