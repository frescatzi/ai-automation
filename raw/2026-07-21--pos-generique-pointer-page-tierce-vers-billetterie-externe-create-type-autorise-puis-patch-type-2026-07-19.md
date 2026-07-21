---
type: raw
title: "POS-GENERIQUE_pointer-page-tierce-vers-billetterie-externe-create-type-autorise-puis-patch-type_2026-07-19"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_pointer-page-tierce-vers-billetterie-externe-create-type-autorise-puis-patch-type_2026-07-19.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Faire pointer une page événement tierce vers une billetterie externe quand l'API refuse le type « externe » à la création

**Date :** 19.07.2026 · **Portée :** runner n8n qui crée/maj une entité sur une plateforme tierce (ex. Wix Events V3) dont le **type de registration/mode** n'est pas modifiable librement à la création. Réutilisable hors AFTRSN.

---

## 1. Problème générique
Une API de création d'entité impose un champ « type » (mode d'inscription, statut initial, variante) dont l'**enum à la création** est plus restreint que l'enum à la mise à jour. On veut un état final (ex. **registration externe** vers un lien tiers) que la création **n'autorise pas**. Envoyer directement la valeur cible provoque une erreur trompeuse (« champ requis manquant » alors qu'on l'a fourni → la valeur est en réalité **hors enum de création** et donc ignorée).

## 2. Règle
**Créer avec un type autorisé, puis PATCH le type vers la valeur cible.**
1. `create` avec la valeur d'enum **autorisée à la création** la plus neutre (ex. `RSVP`), en **draft** si possible.
2. `update`/`PATCH` l'entité pour poser la valeur cible sur le **champ mutable** (souvent un champ « type » distinct du « type initial » figé), avec sa sous-configuration (ex. `external:{url}}`).
3. Ne poser l'état final que **si nécessaire** (garde conditionnelle : ne PATCHer que pour le sous-ensemble concerné — ex. `needsExternal`).

## 3. Points de conception (n8n)
- **Séparer create et post-patch** : le nœud de création reste générique (type autorisé) ; un `IF` conditionne le post-patch au cas cible.
- **Brancher le post-patch APRÈS la chaîne déjà vérifiée** (écriture des références + statut), pour ne pas fragiliser l'existant.
- **Idempotence** : écrire les références (id tiers) **avant** le post-patch → un échec du post-patch ne provoque pas de re-création au tick suivant (l'entité passe alors en route `update`). L'échec reste visible (Error-WF) et l'entité reste un **draft** à valider.
- **Transporter le contexte** : le nœud `create` émet les champs nécessaires au post-patch (id cible, url, drapeau), pour que le PATCH ne dépende pas d'une relecture.

## 4. Leçon transverse de LECTURE (projection d'API)
Un champ **écrit et persisté** peut être **absent d'un GET** si la **projection** (`fields`, `select`, `expand`…) ne l'inclut pas. Un GET partiel qui renvoie « vide » ne prouve **pas** la non-persistance.
- Toujours vérifier avec la **projection complète** avant de conclure « non persisté ».
- Certains couples champ↔projection sont non intuitifs (ex. Wix Events V3 : `TEXTS` → description longue ; `shortDescription` → **`DETAILS`** ; `fields` est un **tableau** cumulable).
- Un **succès HTTP** ne prouve que l'acceptation de la requête, pas le contenu relu : vérifier par **relecture ciblée**, pas par le seul code 200.

## 5. Vérification obligatoire
- Prouver le mécanisme sur un **objet jetable** (create → patch → GET projection complète → delete) **avant** de patcher le runner.
- Re-tester **live** via une exécution réelle (pas un `success` isolé) et lire les **items de sortie** des nœuds clés.
- Contrôler le graphe **publié** (`mode:'active'`) après tout patch.

## 6. Anti-patterns
- Envoyer la valeur d'enum cible à la création « parce qu'elle existe dans l'enum de réponse » (les enums create/update/response diffèrent souvent).
- Conclure « bug d'écriture » sur un GET partiel sans avoir demandé la projection complète.
- Forcer un `create` de test sur une entité déjà avancée (ex. plusieurs éléments amont éligibles) sans neutraliser les surnuméraires → doublons de création.
