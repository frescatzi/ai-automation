---
type: raw
title: "POS-GENERIQUE_refondre-runner-pack-unique-vers-bases-par-canal-upsert-titre_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_refondre-runner-pack-unique-vers-bases-par-canal-upsert-titre_2026-07-19.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Refondre un runner « 1 pack » vers « N bases par canal », avec upsert idempotent par titre

**Date :** 19.07.2026 · **Contexte d'origine :** AFTRSN P3 · refonte M2 (Contents & Metrics) · **Type :** POS-GÉNÉRIQUE (réutilisable).

> Patron : un agent produit un « pack » multi-canaux déposé dans une base fourre-tout ; on veut que chaque canal ait sa propre base (contenu à sa place, métriques à leur place) — sans étape d'éclatement intermédiaire.

---

## 1. Difficultés
1. **La base fourre-tout mélange les responsabilités** (contenu de N canaux + colonnes de suivi) : la vue « par canal » ment, le tracking des chiffres n'a pas de place propre.
2. **L'objet 1-par-entité** (ex. la page web d'une édition, unique quel que soit le nombre de vagues) exige un **upsert** — create à la 1ʳᵉ vague, update ensuite — sinon doublons.
3. **Piège n8n : objet JSON littéral dans une expression `={{ ... }}` → « invalid syntax »** (les `}}` imbriqués cassent le parseur). Sournois : avec `onError: continueRegularOutput`, l'erreur devient un item `{error:…}` qui continue → l'IF croit « pas de résultat » → create systématique → **doublons silencieux**, alors que l'exécution est « verte ».
4. Refondre en place un runner ACTIF sans casser la chaîne existante (branche Blocked, mark-task, garde-fous).

## 2. Solutions (le patron)
1. **Découper l'ancienne base en 2 environnements × N canaux** : `<Canal> Content` (texte + statut + lien publié) et `<Canal> Metrics` (chiffres + As-of, relié DUAL à sa ligne Content + à l'entité). Les colonnes de chaque base = uniquement ce qui concerne son canal (zéro case vide). Relations DUAL vers l'entité → navigation depuis la page entité.
2. **Pas de runner d'éclatement** : refondre le **tail** du runner producteur pour écrire directement N lignes (une par canal) à la place d'une. La tête (scan, éligibilité, agent LLM, garde-fou de conformité JSON) ne bouge pas.
3. **Titres canon** portant entité + sous-clé : `<entité> · <vague> · <Canal>` pour les objets par-vague ; `<entité> · <objet>` pour les objets 1-par-entité. C'est la clé de matching de TOUTE la chaîne aval (jamais la relation — non fiable en scan).
4. **Upsert par titre pour l'objet 1-par-entité** : Code (`JSON.stringify` du filtre titre) → HTTP query → IF `results.length>0` → update (même page) / create. Le statut repasse à `To Validate` à chaque update (re-approbation du nouveau texte).
5. **Règle canonique n8n** : tout body JSON d'un nœud HTTP se construit dans un **nœud Code** et se référence `={{ $json.xxx }}`. Ne JAMAIS écrire d'objet littéral dans l'expression.
6. **Refonte en place, atomique** : `n8n_update_partial_workflow` (removeNode des anciens nœuds + addNode + addConnection, `sourceOutput` 0/1 pour les branches IF) puis `validate_workflow` puis **test live 2 vagues** — vague 1 prouve les creates, vague 2 prouve l'update (même page id) ET la non-régression des lignes par-vague.
7. **Transition** : l'ancienne base devient « Intake » hors structure, encore lue par les runners aval → les **repointer un par un** avant de l'archiver. Jamais de bascule big-bang.

## 3. Lessons
- **« Une ligne par canal » rend l'approbation par canal naturelle** : chaque ligne porte son Status ; le gate aval (publication, push API) filtre `Approved` dans SA base.
- **Vague 2 est le vrai test.** La vague 1 réussit toujours (tout est à créer). C'est la 2ᵉ passe qui prouve l'idempotence — tester les DEUX avant de valider.
- **Une exécution verte ne prouve rien avec `continueRegularOutput`** : inspecter la sortie réelle des nœuds (l'item d'erreur transformé se voit dans `n8n_executions mode:filtered`).
- **Le renommage/déménagement d'une base Notion ne casse pas les runners** (DB/DS ids stables) — c'est ce qui permet la transition douce Intake → archive.
- Conserver les données du test sur l'entité de test permanente (ici EP05) : elles servent de jeu de données au test final de bout en bout.
