---
type: raw
title: "MILESTONE_Cablage-Maestro-Router_2026-07-02"
source_url: "drive:1YQ5b80QxLL2ejSu-u-fKlZ4Z1gH7xm4M"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# MILESTONE — Câblage Maestro↔sub-agents + Router (A1, A2, A6) — 2026-07-02

**Résultat : le cœur du LUMINA AI OS tourne bout en bout en PROD.**
Chat → Maestro → délégation aux sub-agents → routage multi-LLM → synthèse.

## Corrections appliquées et testées

1. **A1 — Tools de Maestro recâblés** : suppression des 3 nodes obsolètes (Channel-Content, Acquisition-Performance, « Brand-Buardian » avec typo) ; ajout de 4 `Call n8n Workflow Tool` → Culture-Steward, Experience-Designer, Secretary, Marketing, pattern `$fromAI('query')`. Test : délégation à l'Experience-Designer validée (verdict bar à jus / rituel).
2. **A2 — Secrétaire réparé** : vrai prompt v2 (draft don't send, agenda, fichiers, voix de marque) ; credential Postgres ajoutée au Chat Memory (cause de l'inactivation) ; renommé **AFTRSN-Secretary** ; activé.
3. **A6 — Maestro → LUMINA-AI-Router câblé** : tool `Call 'LUMINA-AI-Router'` (task_type + instruction + context_raw via `$fromAI`) + section ROUTING dans le prompt de Maestro. **Test bout en bout validé : `task_type research` → `google/gemini-3.5-flash-20260519`.**

## Découvertes techniques (pièges figés)

- **Les alias OpenRouter `~…-latest` fonctionnent** (anti-churn validé en conditions réelles : `~openai/gpt-mini-latest` → `gpt-5.4-mini-20260317`). Fallbacks pinnés tous présents au catalogue (338 modèles).
- **Trigger `executeWorkflowTrigger` en mode jsonExample = filtre silencieux** : tout champ absent de l'exemple JSON est droppé. Symptôme : le sub-workflow reçoit des défauts. Fix : élargir le jsonExample à tous les contrats acceptés.
- Le Router accepte désormais **deux contrats** : à plat (`task_type`, `instruction`, `context_raw` — pour les appels-outils d'agents) et imbriqué (`routing.task_type`, `task_payload.*` — contrat JSON brand-agnostic d'origine). Rétro-compatible.
- API n8n : `PATCH /rest/workflows/:id` (avec versionId) crée un draft ; `POST /rest/workflows/:id/activate` (avec versionId) publie. Publication refusée si un node a une credential manquante.

## État des anomalies

✅ A1 · ✅ A2 · ✅ A6 · ✅ slugs — Restent : **A3** (brancher Hermes comme outil Ops), **A4** (chat memory Experience-Designer), **A5** (prompt Channel-Content à dépoussiérer), **A7** (modèles par rôle via OpenRouter vs gpt-4.1-mini direct — à trancher).

## Prochaine session

1. A3 — Hermes outil Ops (Secrétaire, Marketing) ; 2. A7 — trancher les modèles par rôle ; 3. A4/A5 en passant ; 4. mémoire épisodique + RAG dans le Router.

---
*MILESTONE — 2026-07-02, Claude (Cowork). Réfs : PLAN-PROJET v2.2, MILESTONE audit n8n du même jour, POS EXACT LUMINA-AI-Router 01-07.*
