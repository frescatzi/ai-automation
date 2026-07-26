---
type: raw
title: "POS-AFTRSN_Outreach-Relances-Notion-Drafts_20260704"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_outreach-relances-notion-drafts_20260704.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS AFTRSN — Outreach + relances piloté par Notion, à brouillons (Venue-Coordinator)

Procédure vérifiée le 04.07.2026 — construction, test (4 chemins) et publication du processus `AFTRSN-Venue-Coordinator` (`Su9tO0fS39u9itlU`). Version générique : `POS-GENERIQUE_Outreach-Relances-Base-Drafts_20260704.md`.

## 1. Prérequis AFTRSN

- Base **Venues** 🎭 (`b9001abd-8332-4bbb-9673-6deb5159b4f3`) enrichie le 04.07.2026 : champ `Email` (type email) + champ `Gmail Thread` (texte). Champs de pilotage déjà présents : `Status`, `1st Email`, `Follow-up 1`, `Follow-up 2`, `Contact`, `Notes`, `Capacity`.
- Credential **PARTNERS-GMAIL** (`ohE4nLNpeFYMG9Ha`) : créée par Karter avec le même client OAuth Google que Hello (API Gmail déjà activée sur le projet `909839129866`), connexion au compte Partners (B2B). Règle : venues/artistes = Partners, clients = Hello.
- Agent **AFTRSN-Secretary** (`FtPFOKvi2t18DFb1`), contrat `{query}` ; **LuminaOsBot - Telegram** (chat `776345147`) ; **LUMINA-MEMORY-WRITE** (`zu4jfZbmDz8trQLl`).

## 2. Pattern construit (17 nodes, 1 seul workflow)

Schedule (cron `0 9 * * 1-5`, **settings.timezone = Europe/Zurich** car l'instance est en UTC) → Notion getAll (toute la base, le tri se fait en Code — évite les filtres Notion fragiles) → Code « Actions du jour » (Prospect + email requis ; sans `1st Email` → premier contact ; J+3 sans `Follow-up 1` → relance 1 ; `Follow-up 1` posé et J+7 sans `Follow-up 2` → relance 2 ; sinon ignoré → 0 item = arrêt silencieux voulu) → Secretary (sortie balisée `SUJET:`/`RESUME:`/`---DRAFT---`) → Code « Découper » (parse + repli) → **Switch 3 branches** → par branche : Gmail Create Draft (nouveau fil, ou `options.threadId` pour relancer dans le fil) puis Notion update (la date correspondante ; le premier contact écrit aussi `Gmail Thread` ← `message.threadId` du brouillon) → Merge (append, 3 entrées) → Telegram par lieu (format Karter, zéro code technique) → Code « Compter » → Save Episode → Set Résumé.

**Pourquoi un Switch et non un node unique :** les 3 actions écrivent des propriétés Notion différentes et le node Gmail relance exige un threadId non vide — les clés de propriétés n8n sont statiques, donc une branche par action est la forme la plus lisible (règle « vue simple par étape »).

**Idempotence :** les dates écrites en fin de chaîne empêchent tout re-draft le lendemain ; l'arrêt des relances est le changement de `Status` par l'humain à la première réponse (v1 sans détection automatique).

## 3. Difficultés rencontrées

1. Premier run « Success » mais chaîne morte après le node de sélection : 0 item produit alors que le lieu de test existait.
2. Lecture du détail d'exécution (`includeData=true`) : le décodage du format « flatted » de n8n par un résolveur récursif a gelé l'onglet Chrome (timeouts CDP 45 s en boucle) ; l'exécution suivante, plus grosse (trace d'agent), rendait même le simple fetch impraticable.
3. Clics « Execute workflow » perdus : références DOM périmées après rechargement de page, bouton déplacé par l'ouverture du panneau Logs.
4. Fuseau horaire : cron « 9h » sur une instance en UTC = 11h à Zurich en été.
5. La liste `/rest/credentials` est illisible telle quelle depuis l'outil JS (filtre de sortie « sensitive » du navigateur agentique).

## 4. Solutions implémentées

1. Inspection de la **vraie** sortie du node Notion getAll : format simplifié `property_*` en snake_case (`property_email`, `property_1st_email`…), titre dans `name` — code réécrit contre le format observé.
2. Abandon du parsing des exécutions : **vérification par les effets** dans les systèmes cibles (dates et thread id relus dans Notion via le connecteur, notifications Telegram constatées par Karter) + tests booléens légers (`raw.includes('"NomDuNode"')`) pour savoir quels nodes ont tourné.
3. Re-screenshot avant chaque clic + clic par coordonnées fraîches ; vérification qu'un nouveau run existe (`/rest/executions?limit=1`) avant d'attendre son résultat.
4. `settings.timezone = 'Europe/Zurich'` posé explicitement sur le workflow, puis **re-publication** (le PATCH crée une version non publiée).
5. Récupération des ids de credentials par leurs **références dans les nodes de workflows existants**, ou par une projection minimale (id tronqué + nom) qui passe le filtre.

## 5. Leçons apprises

1. **La sortie « simplifiée » d'un node n8n est un format à part entière** — jamais les noms de propriétés de l'API source. Exécuter une fois, regarder, coder contre l'observé (leçon jumelle du parseur Gmail, toujours vraie).
2. **Vérifier par les effets, pas par les traces** : quand le détail d'exécution est trop lourd ou illisible, relire les systèmes cibles (Notion, Gmail, Telegram) prouve la chaîne de bout en bout mieux qu'un parsing de payload.
3. Un run « Success » ne prouve que le chemin emprunté — tester **chaque branche** d'un Switch en manipulant les données de pilotage (antidater les champs date du test), et tester le chemin à vide.
4. Sur une instance n8n en UTC, tout déclencheur horaire destiné à un humain doit porter un `settings.timezone` explicite — et toute modification post-publication doit être **re-publiée**.
5. Un algorithme récursif copiant son état à chaque branche (Set par nœud) explose combinatoirement sur un graphe réel — dans un onglet piloté par CDP, ça gèle le navigateur de l'utilisateur : préférer des vérifications O(n) simples dans le contexte d'une page.
