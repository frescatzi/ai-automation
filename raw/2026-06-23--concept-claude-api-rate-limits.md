---
type: raw
title: "concept-claude-api-rate-limits"
source_url: "drive:1MpglmFJ5uKinG9hjqKF9MbOmn0-oMoNe"
captured: 2026-06-23
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Limites de débit Claude API — l'essentiel"
status: draft
publish: none
vault: ai-automation
brand: null
sources: [raw/2026-06-22--claude-api-rate-limits.md]
related: ["concept-prompt-caching.md", "concept-gestion-erreurs-429.md"]
updated: 2026-06-22
---

# Limites de débit Claude API — l'essentiel

Les limites encadrent l'usage de l'API par **organisation**, sur deux axes : **dépenses** (coût mensuel max) et **débit** (requêtes/tokens par minute). Elles montent **automatiquement** de niveau selon les seuils de dépense atteints.

## Les 3 limites de débit qui comptent

Mesurées **par modèle**, par minute :

- **RPM** — requêtes par minute
- **ITPM** — tokens d'entrée par minute
- **OTPM** — tokens de sortie par minute

Dépassement → **erreur `429`** + en-tête `retry-after` (secondes à attendre). Algorithme **seau à jetons** : la capacité se réapprovisionne en continu (pas de reset fixe), donc une rafale courte peut déclencher un 429 même sous la limite « par minute ».

## Le levier n°1 : le prompt caching

Point clé sous-estimé : pour la plupart des modèles, **les tokens lus depuis le cache (`cache_read_input_tokens`) ne comptent PAS dans l'ITPM** (et coûtent 10 % du prix de base).

→ En cachant le contenu répété (instructions système, gros contextes, définitions d'outils, historique), on **multiplie le débit effectif** sans changer la limite. *Exemple : ITPM de 2 M + 80 % de cache ≈ 10 M tokens d'entrée/min traités.*

Voir [[concept-prompt-caching]].

## À retenir pour construire un agent robuste

1. **Gérer les 429** : lire `retry-after`, faire un backoff, ne pas marteler. Voir [[concept-gestion-erreurs-429]].
2. **`max_tokens` ne pénalise pas l'OTPM** — le mettre haut sans crainte.
3. **Limites par modèle** → on peut saturer plusieurs modèles en parallèle (Opus = limite combinée 4.x ; Sonnet = combinée 4.x).
4. **Monter en charge progressivement** pour éviter les limites d'accélération sur les pics.
5. **Surveiller** : Console → page *Usage* (graphes input/output + taux de cache).

## Implication pour le projet Lumina

Le robot d'ingestion utilise **Gemini** (gratuit) pour le tri → pas de pression sur les limites Claude. Si un jour on bascule la rédaction `wiki/` ou la mémoire agents sur l'API Claude, **activer le prompt caching** sur les instructions système (le `CLAUDE.md` / `ROUTING.md` répétés à chaque appel) sera le premier réflexe pour tenir le débit.

---

> Source : `raw/2026-06-22--claude-api-rate-limits.md` (capturé depuis la doc Anthropic, 2026-06-22).
