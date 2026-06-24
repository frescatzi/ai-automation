---
type: wiki
title: Limites de débit et de dépenses — API Claude
status: active
publish: none
vault: ai-automation
brand:
sources:
  - raw/2026-06-22--claude-api-rate-limits.md
related:
  - wiki/concept-prompt-caching.md
updated: 2026-06-22
---

# Limites de débit et de dépenses — API Claude

## Idée centrale

L'API Claude encadre l'usage par organisation via deux mécanismes indépendants : une **limite de dépenses** (plafond mensuel en $) et une **limite de débit** (RPM / ITPM / OTPM, par modèle). Les deux montent automatiquement de niveau selon des seuils d'achat de crédits ou d'usage — ce ne sont jamais des minimums garantis, seulement des maximums autorisés.

## Points utiles

- **Seau à jetons (token bucket)** : la capacité se réapprovisionne en continu plutôt que de se réinitialiser à intervalle fixe. Conséquence pratique : de courtes rafales de requêtes peuvent déclencher une erreur **429** même si le débit moyen reste sous la limite.
- **Limites appliquées par modèle**, séparément (Opus 4.x regroupe 4.8/4.7/4.6/4.5/4.1/4 ; Sonnet 4.x regroupe 4.6/4.5/4 — voir `raw/2026-06-22--claude-api-rate-limits.md`).
- **Le cache de prompt n'est quasiment pas compté dans l'ITPM** (sauf Haiku 3.5) → voir [[concept-prompt-caching]] pour le mécanisme et l'impact sur le débit effectif.
- `max_tokens` n'affecte pas l'OTPM → aucune raison de le sous-dimensionner.
- Erreur 429 → en-tête `retry-after` (secondes avant retry) + en-têtes `anthropic-ratelimit-*` exposant limite/restant/reset, **toujours la valeur la plus restrictive** (org vs espace de travail).
- Niveaux de dépense distincts des niveaux de débit, mais le passage de niveau de dépense (achat de crédits) déclenche aussi le déblocage de débits plus élevés.
- Cas particuliers à limites propres : **Message Batches** (50 RPM, files jusqu'à 100k requêtes) et **Managed Agents** (300 req/min création, 600 req/min lecture).
- Mode `speed: "fast"` (Opus 4.8/4.7/4.6) a ses propres limites, indépendantes des limites Opus standard.

## Limites / conditions d'usage

- Synthèse valable pour les niveaux 1 (limites standard documentées) ; les niveaux 2-4 et le custom ont des plafonds plus élevés non détaillés ici — voir Console → Limits pour les chiffres à jour.
- Les limites par espace de travail héritent de celles de l'organisation si non définies explicitement ; l'org reste toujours la limite plafond.

## Voir aussi

- [[concept-prompt-caching]] — comment le cache réduit le coût ITPM effectif.


<!-- test déclencheur croisé 2026-06-24 -->
