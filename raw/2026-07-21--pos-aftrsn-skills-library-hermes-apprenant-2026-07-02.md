---
type: raw
title: "POS-AFTRSN_Skills-library-Hermes-apprenant_2026-07-02"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_skills-library-hermes-apprenant_2026-07-02.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# AFTRSN-LUMINA — POS EXACT — Hermes apprenant : bibliothèque de skills + maturité — 2026-07-02

Hermes consulte des runbooks réutilisables avant d'agir, journalise ses usages, et les skills gagnent en autonomie automatiquement à mesure qu'ils font leurs preuves.

## 0. Références

| Élément | Valeur |
|---|---|
| Registre | table `skill_registry` (`skill_slug` PK, `brand`, `title`, `maturity`, `uses`, `successes`, `failures`, `created_at`, `updated_at`) |
| Contenu des skills | `collection='skills'` dans `lumina_memory` (génériques) + `aftrsn_memory` (marque) |
| Consommation | `LUMINA-Hermes-Exec` (`Dwv4rcMqNAyQzlrF`) |
| Auto-promotion | `LUMINA-MEMORY-CONSOLIDATION/NIGHTLY` (`dVZyCwGYqnqMpS3P`) |
| Console SQL utilitaire | `__tmp_skill_registry_ddl` (`w9XINCretyHOnBVZ`) — à renommer/nettoyer |

## 1. Modèle

- Un skill = un runbook (but + étapes + garde-fous) stocké comme n'importe quelle entrée mémoire, `collection='skills'`, `source_ref = slug` (kebab-case, préfixe `skill-`).
- Les **stats + maturité** vivent dans une table à part `skill_registry` (le contenu vectoriel n'est pas ré-écrit à chaque graduation).
- Maturité : `supervised` (guidage, ne fait pas) → `autonomous` (peut exécuter les étapes NON critiques). **Actions critiques (dépense/envoi/publication/suppression) toujours gatées par Karter, quel que soit le niveau.**

## 2. Consommation (Hermes-Exec)

Chaîne : Prep (extrait le message) → Embed Task (OpenAI) → Build Skills Query → **Skills Search** (Postgres : UNION des 2 banques `JOIN skill_registry ON skill_slug=source_ref WHERE collection='skills' AND maturity IN ('supervised','autonomous') ORDER BY embedding <=> vec LIMIT 2` par banque) → Hermes Login → **Hermes Exec** (préfixe les skills au message, avec leur `[maturité]` ; poll ≤14) → **Build Log** → **Log Skills** (Postgres) → Result.
- Qualifier les colonnes dans le JOIN (`m.title` etc.) sinon *« column reference title is ambiguous »*.
- **Log** : succès → `uses+1, successes+1` + graduation ; échec → `uses+1, failures+1, maturity='supervised'`.

## 3. Graduation auto (inline, choix Karter « vite et bien »)

Dans Build Log (branche succès) :
```
UPDATE skill_registry SET maturity='autonomous', updated_at=now()
WHERE maturity='supervised' AND uses>=3 AND (successes::float/NULLIF(uses,0))>=0.9 AND skill_slug = ANY(ARRAY[...]);
```
Rétrogradation (branche échec) : `maturity='supervised'` immédiat. Pas de confirmation manuelle (apprentissage rapide) ; le filet = gating des actions critiques + rétrogradation au 1er échec.

## 4. Auto-promotion nocturne (création 100 % auto)

Branche ajoutée à la consolidation : Get Episodes → Build Skills Prompt → **Skills LLM** (OpenRouter, sortie JSON stricte `{slug,title,brand,content}` ou `{slug:null}`) → Parse Skill → Embed Skill → Build Skill Insert (INSERT memory `collection='skills'` **+** INSERT `skill_registry` maturity=`supervised`, params `$1..$7`, `ON CONFLICT DO NOTHING`) → Insert Skill+Registry (Postgres). 1 skill/nuit. Testé : `skill-theme-after-sun-validation` auto-créé.

## 5. Pièges figés

1. **Task runner ~60s** sur les nodes Code → polling Hermes plafonné (14×3s ≈ 45s) pour ne pas timeout (« Task execution timed out »).
2. **JOIN → colonnes ambiguës** : qualifier (`m.title`, `r.maturity`).
3. `ANY(ARRAY['a','b'])` OK pour matcher plusieurs slugs ; multi-UPDATE dans un seul executeQuery OK.
4. Pas de seuil de similarité → un skill peu pertinent remonté dans le top-2 est quand même compté (amélioration future).
5. Auto-graduation nécessitait des compteurs → d'où `skill_registry` (le schéma des tables mémoire est figé).

---
*POS EXACT — 2026-07-02, Claude (Cowork). Voir : POS-GENERIQUE bibliothèque de skills apprenante ; POS mémoire vivante ; POS Hermes outil Ops.*
