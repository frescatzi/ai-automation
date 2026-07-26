---
type: raw
title: "POS-GENERIQUE_Bibliotheque-de-skills-apprenante-agents"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_bibliotheque-de-skills-apprenante-agents.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Bibliothèque de skills apprenante pour agents (maturité + graduation)

Patron réutilisable : donner à un exécuteur (agent Ops) des runbooks réutilisables qu'il consulte avant d'agir, avec des niveaux de maturité qui progressent automatiquement selon les résultats. S'appuie sur une mémoire vectorielle + une petite table registre.

## 1. Deux stockages complémentaires

- **Contenu du skill** dans la banque vectorielle, `collection='skills'` (générique dans une banque partagée + spécifique dans la banque marque). `source_ref = slug`.
- **Stats + maturité** dans une table registre à part (`skill_slug, brand, title, maturity, uses, successes, failures, created_at`). Raison : la maturité change souvent ; on ne veut pas ré-embedder le skill à chaque évolution. (Le schéma vectoriel reste figé.)

## 2. Boucle d'apprentissage

1. **Consulter** : avant d'exécuter, l'agent embed sa tâche → recherche les skills pertinents (banques + `JOIN registry`, maturité ≥ supervised) → les préfixe à son prompt (avec le niveau de chaque skill).
2. **Journaliser** : après exécution, incrémenter `uses` (+`successes` si OK, +`failures` sinon) pour les skills utilisés.
3. **Graduer** : `supervised → autonomous` quand un seuil est atteint (ex. `uses≥3` et taux de succès `≥90%`). Rétrograder à `supervised` au premier échec.
4. **Créer** : soit un outil « Save Skill » manuel (contrôlé), soit une auto-promotion nocturne (un LLM extrait les procédures récurrentes des épisodes) — ou les deux.

## 3. Sémantique de maturité

- **supervised** = guidage : l'agent propose, n'exécute pas.
- **autonomous** = l'agent exécute les étapes **non critiques** sans redemander.
- **Invariant absolu** : les actions critiques (dépense, envoi, publication, suppression) restent **toujours** gatées par l'humain, quel que soit le niveau. C'est ce qui rend une graduation rapide acceptable.

## 4. Arbitrage vitesse ↔ sûreté

Seuil bas (graduation rapide) = plus de levier, plus tôt ; le filet = gating des actions critiques + rétrogradation immédiate au moindre échec. Seuil haut / confirmation manuelle = plus sûr, moins de levier. Choisir selon la tolérance au risque du domaine.

## 5. Pièges figés

1. **Timeout du task runner** (~60s sur les nodes Code) → plafonner les boucles de polling.
2. **Colonnes ambiguës** dans les JOIN → toujours qualifier (`m.col`, `r.col`).
3. **Pas de seuil de similarité** = des skills peu pertinents remontés sont comptés → ajouter un seuil de distance.
4. **Auto-promotion** : imposer une sortie LLM en **JSON strict** + parsing défensif (sinon rien ne s'insère) ; 1 skill/exécution pour borner le volume ; idempotence via `md5(content)` et `ON CONFLICT (skill_slug) DO NOTHING`.
5. Séparer **contenu** (vectoriel) et **stats** (registre) pour éviter de ré-embedder à chaque graduation.

## 6. Tester

Insérer 1-2 skills (contenu + registre) → donner à l'agent une tâche qui les concerne → vérifier qu'ils sont préfixés et que `uses/successes` s'incrémentent → simuler le seuil → confirmer le passage `autonomous` ; lancer l'auto-promotion → vérifier qu'un skill naît des épisodes.

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-AFTRSN Skills library Hermes apprenant. Brique centrale du LUMINA-PLAYBOOK.*
