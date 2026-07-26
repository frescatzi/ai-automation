---
type: raw
title: "MILESTONE_Agents-AFTRSN-etat-reel-n8n_2026-07-02"
source_url: "drive:17GcgChvnxm0Ee8c2u1HIR2cx-ElhM65r"
captured: 2026-07-06
vault: brands
brand: AFTRSN
immutable: true
---

# MILESTONE — État RÉEL des agents AFTRSN dans n8n
**Date :** 2026-07-02 · **Source :** lecture directe des workflows n8n (API interne, session Chrome) — *fait foi sur la doc antérieure*
**Contexte :** nouvelle stratégie agents (reconfiguration post-passation du 02-07). n8n est plus à jour que la documentation.

---

## 1. Vue d'ensemble

Dossier n8n : `Personal / 02-AFTRSN / AFTRSN-AGENTS` (+ sous-dossier `Sub-Agents`).

| Agent | Workflow id | Actif | Maj | Modèle | Mémoire chat | Rôle réel (system-prompt) |
|---|---|---|---|---|---|---|
| **Maestro** | `OlrQO21u178SjgBK` | ✅ | 01-07 18:48 | gpt-4.1-mini | Postgres | Orchestrateur, point d'entrée unique, expert curation/event mgmt, synthèse pour Karter |
| **Channel-Content** | `MWUwLi3vu2Ws65lL` | ✅ | 29-06 | gpt-4.1-mini | Postgres | Contenu multi-canaux : Insta, newsletters, site, calendriers éditoriaux |
| **Culture-Steward** | `vK6B2WuDfbse41Jv` | ✅ | 01-07 19:20 | gpt-4.1-mini | Postgres | CHALLENGER crédibilité culturelle (culture électronique africaine, diaspora, anti-clichés) |
| **Marketing** | `wyUEojwZjSKFRaRL` | ✅ | 01-07 19:27 | gpt-4.1-mini | Postgres | Marketing events + **garde-voix de marque** (compliance voix + `forbidden_words`) |
| **Secrétaire** | `FtPFOKvi2t18DFb1` | ❌ inactif | 01-07 19:50 | gpt-4.1-mini | Postgres | ⚠️ prompt erroné (voir anomalie A2) |
| **Experience-Designer** | `bBu5ycRwHoK7ACdV` | ✅ | 01-07 21:03 | gpt-4.1-mini | **aucune** | CHALLENGER design expérientiel du « Day & Night Ritual » |

Tous ont : chatTrigger + executeWorkflowTrigger (appelables par Maestro) + outil **Knowledge** (httpRequestTool → mémoire vectorielle ; Maestro = mémoire LUMINA, les autres = AFTRSN).

## 2. Points conformes à la nouvelle stratégie

- Prompts réécrits en anglais, structurés (ROLE / MEMORY / TOOLS / OUTPUT / NAMING), réponse dans la langue de l'utilisateur (défaut français).
- Karter (Katel) = décideur final partout ; actions critiques (dépense, envoi, publication, suppression) à faire valider explicitement.
- Règle de nommage partout : AFTER SUN PEOPLE (AFTRSN) = marque ; « After Sun » = l'événement ; jamais « ASP » / « Afterson People ».
- Marketing porte bien le rôle de garde-voix (décision du 01-07) ; Culture-Steward et Experience-Designer sont des challengers (pas de livrables finaux).
- Channel-Content confirmé dans son nouveau rôle contenu réseaux sociaux + newsletter (réintégré alors qu'il était prévu 2ᵉ vague).

## 3. ⚠️ Anomalies détectées (à corriger)

- **A1 — Outils de Maestro pas à jour.** Son prompt annonce Culture Steward, Experience Designer, Marketing, Secrétaire, mais ses tool-nodes appellent encore **Channel-Content, Acquisition-Performance et Brand-Guardian** (ancien socle). En plus, le node Brand-Guardian pointe vers « AFTRSN-Brand-**Buardian** » (typo → workflow probablement introuvable). À recâbler vers les 5 agents actuels.
- **A2 — Secrétaire : mauvais prompt.** « Experience / Secrétaire » contient une **copie du prompt Marketing** (erreur de copier-coller) et le workflow est **inactif**. À réécrire + activer.
- **A3 — Hermes non branché.** Le prompt Marketing dit « Use Hermes for web/market research », mais aucun node Hermes n'existe dans le workflow. À brancher (étape 5 de la passation).
- **A4 — Experience-Designer sans mémoire de chat.** Pas de node Postgres Chat Memory, contrairement aux autres. À ajouter si voulu.
- **A5 — Prompt Channel-Content obsolète.** Il référence encore Brand-Guardian et Acquisition-Performance comme rôles actifs (agents transformés depuis). Mise à jour mineure.
- **A6 — Router non câblé.** Tous les agents utilisent directement `gpt-4.1-mini` (node OpenAI). Aucun appel à `LUMINA-AI-Router` (`Cu8SozYmondKM8RB`) — le test bout en bout Maestro → Router reste à faire (étape 2 de la passation).

## 4. Écarts doc ↔ réalité (pour mémoire)

- La passation du 02-07 décrivait Brand-Guardian/Acquisition-Performance comme « socle existant » : ils ont été **transformés/renommés** en agents de la nouvelle vague (confirmé par Karter) ; seuls les tool-nodes de Maestro y font encore référence (A1).
- Les « system-prompts v2 à valider » : une version retravaillée est déjà **câblée en PROD** dans n8n. La référence = n8n, pas le doc v2.

## 5. Prochaines actions recommandées (ordre)

1. Corriger **A1** (recâbler les tools de Maestro vers les 5 agents actuels, supprimer la typo Brand-Buardian).
2. Corriger **A2** (vrai prompt Secrétaire + activation).
3. Câbler **Maestro → LUMINA-AI-Router** (A6) = test de routage bout en bout.
4. Brancher **Hermes** comme outil Ops (A3), puis A4/A5 en passant.

---

*MILESTONE v1.0 — rédigé le 2026-07-02 par Claude (Cowork) à partir des workflows n8n live.*
