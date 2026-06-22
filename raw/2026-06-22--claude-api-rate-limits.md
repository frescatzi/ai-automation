---
type: raw
title: "claude-api-rate-limits"
source_url: "drive:1oUVmCaZR6Ob4z7GRpo6IW-PeeFW498jX"
captured: 2026-06-22
vault: ai-automation
brand: null
immutable: true
---

# Claude API — Limites de débit (Rate Limits)

> Source : https://platform.claude.com/docs/fr/api/rate-limits
> Réf. Anthropic — capturé le 2026-06-22.

Pour limiter les abus et gérer la capacité de l'API, des limites encadrent l'utilisation de l'API Claude par organisation. Deux types :

- **Limites de dépenses** : coût mensuel maximal qu'une organisation peut engager.
- **Limites de débit** : nombre maximal de requêtes API sur une période donnée.

Les limites sont appliquées au niveau de l'organisation (et configurables par espace de travail). Elles valent pour le niveau Standard et Priority.

## À propos des limites de débit

- Définies par **niveau d'usage** ; passage **automatique** au niveau supérieur en atteignant certains seuils.
- Appliquées sur des intervalles courts (60 RPM ≈ 1 req/s) → de courtes rafales peuvent déclencher des erreurs.
- Algorithme du **seau à jetons** : capacité réapprovisionnée en continu, pas de reset à intervalle fixe.
- Ce sont des **maximums autorisés**, pas des minimums garantis.

## Limites de dépenses — conditions de passage de niveau

| Niveau | Achat de crédits requis | Achat max (1 transaction) | Limite de dépenses mensuelle |
|---|---|---|---|
| Niveau 1 | 5 $ | 500 $ | 500 $ |
| Niveau 2 | 40 $ | 500 $ | 500 $ |
| Niveau 3 | 200 $ | 1 000 $ | 1 000 $ |
| Niveau 4 | 400 $ | 200 000 $ | 200 000 $ |
| Facturation mensuelle | N/A | N/A | Aucune limite |

- **Limite définie par le client** : ajustable dans Console → Settings > Limits (ne peut pas dépasser le plafond du niveau).
- **Plafond imposé par le niveau** : au-delà du niveau 4 (200 000 $/mois) → Contact Sales. La facturation mensuelle supprime le plafond (paiement net 30 j).

## Limites de débit (API Messages)

Mesurées en **RPM** (requêtes/min), **ITPM** (tokens d'entrée/min) et **OTPM** (tokens de sortie/min), par classe de modèle. Dépassement → **erreur 429** + en-tête `retry-after`. Limites appliquées **séparément par modèle**.

### Niveau 1 (limites standard)

| Modèle | RPM | ITPM | OTPM |
|---|---|---|---|
| Claude Fable 5 | 50 | 100 000 | 20 000 |
| Claude Sonnet 4.x ** | 50 | 30 000 | 8 000 |
| Claude Haiku 4.5 | 50 | 50 000 | 10 000 |
| Claude Haiku 3.5 † (retiré sauf Bedrock/Vertex) | 50 | 50 000 | 10 000 |
| Claude Opus 4.x * | 50 | 500 000 | 80 000 |

- `*` Opus = limite **totale** combinant Opus 4.8 / 4.7 / 4.6 / 4.5 / 4.1 / 4.
- `**` Sonnet 4.x = limite totale combinant Sonnet 4.6 / 4.5 / 4.
- `†` compte aussi `cache_read_input_tokens` dans l'ITPM.

*(Les niveaux 2-4 et personnalisé offrent des limites plus élevées — voir la Console.)*

### ITPM tenant compte du cache (avantage clé)

Pour la plupart des modèles Claude, **seuls les tokens d'entrée non mis en cache comptent dans l'ITPM** :

- `input_tokens` (après le dernier point de rupture de cache) → ✓ comptent
- `cache_creation_input_tokens` (écriture cache) → ✓ comptent
- `cache_read_input_tokens` (lecture cache) → ✗ **ne comptent pas** (sauf Haiku 3.5 †), et facturés à 10 % du prix de base.

Donc `total_input_tokens = cache_read + cache_creation + input_tokens`.

**Exemple :** limite ITPM 2 M + taux de cache 80 % → ~10 M tokens d'entrée traités/min (2 M non-cache + 8 M cache). → **La mise en cache des prompts augmente fortement le débit effectif** (instructions système, gros documents de contexte, définitions d'outils, historique de conversation).

`max_tokens` n'entre **pas** dans le calcul OTPM → aucun inconvénient à le mettre haut.

## API Message Batches (limites propres, partagées entre modèles)

| RPM max | Requêtes de lot max en file | Requêtes de lot max par lot |
|---|---|---|
| 50 | 100 000 | 100 000 |

## Managed Agents (limites par organisation, distinctes)

| Opération | Limite |
|---|---|
| Endpoints de création (agents, sessions, environnements) | 300 req/min |
| Endpoints de lecture (récupération, liste, flux) | 600 req/min |

## Mode rapide (fast)

Avec `speed: "fast"` sur Opus 4.8 / 4.7 / 4.6 → limites dédiées, distinctes des limites Opus standard. Dépassement → 429 + `retry-after`. En-têtes `anthropic-fast-*` indiquent l'état.

## En-têtes de réponse (extraits)

- `retry-after` : secondes à attendre avant de réessayer.
- `anthropic-ratelimit-requests-limit / -remaining / -reset`
- `anthropic-ratelimit-input-tokens-limit / -remaining / -reset`
- `anthropic-ratelimit-output-tokens-limit / -remaining / -reset`
- `anthropic-priority-*` (niveau Priority uniquement)

Les en-têtes `tokens-*` affichent la **limite la plus restrictive** en vigueur (ex. limite d'espace de travail si dépassée).

## Limites par espace de travail

Possibilité de définir des limites de dépenses/débit **par espace de travail** (sauf l'espace par défaut). Si non définies → égales à celles de l'organisation. Les limites org s'appliquent toujours. Lecture programmatique via l'**API Rate Limits**.

## Surveillance

Console → page **Usage** : graphiques RPM/tokens + 2 graphiques de limites de débit (Input / Output Tokens), incluant le taux de cache. Page **Limits** pour consulter/ajuster.
