---
type: wiki
title: "Concept — Bibliothèque de skills apprenante (maturité & graduation)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-07-06--pos-generique-bibliotheque-de-skills-apprenante-agents.md"
  - "raw/2026-07-20--pos-generique-bibliotheque-de-skills-apprenante-agents.md"
related:
  - "concept-hermes-agent"
  - "concept-memoire-vivante-agents"
  - "sop/sop-clonage-roster-agents"
  - "synthese-lumina-ai-os"
  - "sop/sop-apprendre-skill-a-hermes"
updated: 2026-07-20
---

# Concept — Bibliothèque de skills apprenante (maturité & graduation)

> **Idée centrale :** donner à un agent exécuteur des *runbooks réutilisables* qu'il consulte **avant d'agir**, dont le niveau de maturité **progresse automatiquement** selon les résultats. Brique centrale du LUMINA-PLAYBOOK ; complète la [[concept-memoire-vivante-agents]] (épisodique) par une mémoire **procédurale**.

## Deux stockages complémentaires

Un skill vit dans **deux endroits distincts**, et c'est volontaire :

- **Contenu** → banque vectorielle, `collection='skills'` (générique dans la banque partagée + spécifique dans la banque de marque), `source_ref = slug`. Recherché par le sens.
- **Stats + maturité** → table registre à part (`skill_slug, brand, title, maturity, uses, successes, failures, created_at`).

Pourquoi séparer : la maturité change souvent ; garder les stats hors du vecteur évite de **ré-embedder** le skill à chaque évolution (le schéma vectoriel reste figé, cf. [[concept-memoire-vectorielle-multi-marques]]).

## Boucle d'apprentissage

1. **Consulter** — avant d'exécuter, l'agent embed sa tâche → cherche les skills pertinents (banques + `JOIN registry`, maturité ≥ `supervised`) → les préfixe à son prompt avec leur niveau.
2. **Journaliser** — après exécution, incrémenter `uses` (+`successes` si OK, +`failures` sinon) sur les skills utilisés.
3. **Graduer** — `supervised → autonomous` au franchissement d'un seuil (ex. `uses≥3` et succès `≥90%`) ; **rétrograder** à `supervised` au premier échec.
4. **Créer** — via un outil « Save Skill » manuel (contrôlé) et/ou une **auto-promotion nocturne** (un LLM extrait les procédures récurrentes des épisodes).

## Sémantique de maturité

- **supervised** = l'agent *propose*, n'exécute pas.
- **autonomous** = l'agent exécute seul les étapes **non critiques**.
- **Invariant absolu** : les actions critiques (dépense, envoi, publication, suppression) restent **toujours** gatées par l'humain, quel que soit le niveau. C'est ce garde-fou qui rend une graduation rapide acceptable.

## Arbitrage vitesse ↔ sûreté

Seuil **bas** (graduation rapide) = plus de levier, plus tôt — le filet étant le gating des actions critiques + la rétrogradation immédiate au moindre échec. Seuil **haut** / confirmation manuelle = plus sûr, moins de levier. À calibrer selon la tolérance au risque du domaine.

## Pièges figés

| Piège | Correctif |
|---|---|
| Timeout du task runner (~60 s sur les nodes Code) | Plafonner les boucles de polling |
| Colonnes ambiguës dans les `JOIN` | Toujours qualifier (`m.col`, `r.col`) |
| Pas de seuil de similarité → skills hors-sujet comptés | Ajouter un seuil de distance |
| Auto-promotion qui n'insère rien | Sortie LLM en **JSON strict** + parsing défensif ; 1 skill/exécution ; idempotence `md5(content)` + `ON CONFLICT (skill_slug) DO NOTHING` |
| Ré-embedding à chaque graduation | Séparer **contenu** (vectoriel) et **stats** (registre) |

## Tester

Insérer 1-2 skills (contenu + registre) → confier à l'agent une tâche qui les concerne → vérifier qu'ils sont préfixés et que `uses/successes` s'incrémentent → simuler le seuil → confirmer le passage `autonomous` → lancer l'auto-promotion et vérifier qu'un skill naît des épisodes.

## Voir aussi

- [[concept-hermes-agent]] — l'exécuteur concret (Hermes) qui consulte, applique et gradue ces skills à chaque tâche.
- [[concept-memoire-vivante-agents]] — mémoire épisodique + consolidation (le pendant « expérience » de cette mémoire « procédure »).
- [[sop/sop-clonage-roster-agents]] — réplication d'un roster d'agents par marque (les skills spécifiques suivent la marque).
- [[synthese-lumina-ai-os]] — place de la bibliothèque de skills dans le PLAYBOOK et l'AI OS.
- [[sop/sop-apprendre-skill-a-hermes]] — procédure concrète « créer » (étape 4 de la boucle) : écrire un skill à la main dans la bibliothèque et vérifier sa trouvabilité.

*Re-capturé le 2026-07-20 (même patron, corps identique à la source du 2026-07-06) — aucun changement de contenu.*
