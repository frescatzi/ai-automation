---
type: raw
title: "POS-GENERIQUE_Outreach-Relances-Base-Drafts_20260704"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_outreach-relances-base-drafts_20260704.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Processus d'outreach + relances piloté par une base de données, à brouillons uniquement

Pattern vérifié le 04.07.2026 (première application : coordination de lieux événementiels). Réutilisable pour toute campagne d'outreach B2B gérée via LUMINA OS : lieux, artistes, partenaires, sponsors, prestataires. Version spécifique : `POS-AFTRSN_Outreach-Relances-Notion-Drafts_20260704.md`.

## 1. Principe

Une base de données (Notion ou équivalent) est la **source de vérité du pipeline** ; le workflow n'a aucun état propre. Chaque cible porte : un statut (seul `Prospect` est dans le circuit), une adresse email, trois champs date (`premier email`, `relance 1`, `relance 2`) et un champ `thread id`. Le workflow tourne à heure fixe, calcule l'action due par cible, fait rédiger un agent, dépose des **brouillons** (jamais d'envoi automatique — action critique = validation humaine), écrit les dates en retour et notifie l'humain. **L'humain arrête les relances en changeant le statut** dès qu'une réponse arrive — mécanisme volontairement manuel en v1, robuste et sans faux positifs.

## 2. Architecture (17 nodes, 1 workflow)

1. **Schedule** — cron jours ouvrés, **timezone explicite** sur le workflow (les instances sont souvent en UTC).
2. **Lecture de la base** — getAll sans filtre natif (les filtres des connecteurs sont fragiles), tri en Code.
3. **Code « sélection »** — règles : sans date de premier email → `premier_contact` ; premier email ≥ J+3 sans relance 1 → `relance_1` ; relance 1 posée et premier email ≥ J+7 sans relance 2 → `relance_2` ; sinon ignoré. 0 item → arrêt silencieux (comportement voulu, à documenter).
4. **Agent rédacteur** — sortie balisée (`SUJET:` / `RESUME:` / `---DRAFT---`), règles produit dans le prompt (langue, ton, interdiction d'engagement ferme, signature). Le résumé sert la notification humaine.
5. **Code « découpage »** — parse les balises, repli sur texte brut si absentes.
6. **Switch par action** → par branche : **création du brouillon** (nouveau fil, ou dans le fil existant via thread id) puis **écriture en retour dans la base** (la date de l'action ; le premier contact capture aussi le thread id renvoyé par la création du brouillon). Une branche par action car les clés de propriétés sont statiques et le paramètre threadId ne tolère pas le vide.
7. **Merge (append)** → **notification par cible** (aucun jargon technique — exigence produit) → compteur → épisode mémoire → résumé.

## 3. Invariants du pattern

- Draft-only : l'envoi est toujours un geste humain.
- Idempotence par les dates : une action due n'est produite qu'une fois, le re-run du même jour ne produit rien.
- L'ordre brouillon → écriture base signifie qu'un crash entre les deux peut produire un brouillon orphelin re-généré au run suivant — bénin en draft-only, à connaître.
- Les relances se calculent sur la date du **brouillon**, pas de l'envoi réel : si l'humain envoie tardivement, il recale la date de premier email dans la base.
- Tester les **quatre chemins** avant publication : premier contact, relance 1, relance 2 (en antidatant les champs date d'une fiche de test jetable), et le chemin à vide.

## 4. Difficultés rencontrées (lors de la première application)

1. Chaîne morte silencieuse au premier run : le code de sélection était écrit contre un format supposé de la sortie du connecteur base de données.
2. Détail d'exécution illisible : le format interne (« flatted ») décodé récursivement a gelé l'onglet navigateur ; les exécutions incluant une trace d'agent sont trop lourdes à récupérer.
3. Clics UI perdus après rechargements de page (références périmées).
4. Déclencheur décalé de 2 h : instance en UTC, cron pensé en heure locale.

## 5. Solutions implémentées

1. Exécuter une fois, inspecter la vraie sortie du connecteur (format simplifié aux noms transformés), coder contre l'observé.
2. **Vérification par les effets** : relire les systèmes cibles (base, boîte email, messagerie de notification) au lieu de parser les traces d'exécution ; tests booléens légers (présence du nom d'un node dans le payload brut) pour tracer grossièrement.
3. Re-screenshot avant chaque clic, et vérifier qu'un nouveau run est bien apparu avant d'en attendre le résultat.
4. Timezone explicite dans les settings du workflow + re-publication après tout PATCH.

## 6. Leçons apprises

1. La sortie « simplifiée » d'un connecteur est un format à part entière — ne jamais transposer les noms de champs de l'API source.
2. Quand la télémétrie est plus lourde que le système, vérifier le système : les effets dans les outils cibles sont la preuve de bout en bout.
3. Un « Success » ne couvre que le chemin emprunté : chaque branche d'un aiguillage se teste en manipulant les données de pilotage.
4. Tout déclencheur horaire destiné à un humain porte une timezone explicite.
5. Le pipeline appartient à la base de données, pas au workflow : c'est ce qui rend le processus lisible, reprenable à la main, et migrable (ex. vers une bibliothèque de runbooks d'exécution type Hermès).
