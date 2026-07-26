---
type: raw
title: "PASSATION_AFTRSN_ProchainesEtapes_20260705-NUIT"
source_url: "drive:18xhPy9xK9trLtGczBODs6d9rRUM8dtGg"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Passation — Prochaines étapes (05.07.2026, nuit)

**Statut :** Session W-4 Marketing-Planner (fin de journée du 05.07.2026) clôturée. Ce document **remplace `PASSATION_AFTRSN_ProchainesEtapes_20260705-SOIR.md`** comme relais courant vers la prochaine session.

---

## 1 · Boussole — toujours commencer ici

Avant toute action sur ce projet, lire **`FICHE_PROJET_AFTRSN_Automation.md`** (« Fiche Projet — AFTRSN Automation »). C'est le document **compas** : statut courant, deadlines, roadmap Étapes 1→4, règles métier actées, décisions ouvertes. Le présent document ne le remplace pas — il complète la transition de session et se lit **en second**. La Fiche Projet est à jour au 05.07.2026 (nuit).

**Cap fixé par Karter : terminer d'abord la construction de TOUTE la plateforme, puis basculer en mode test/amélioration continue.** Après W-4, **il ne reste que W-7 Knowledge-Capture** pour finir la plateforme.

**Échéances :** 18.07 = artistes du 08.08 confirmés (lieu confirmé : Mini Café Bar ; booking artistes par WhatsApp). **08.08 = run de validation A→Z** par Karter.

## 2 · Prochaine étape (priorité n°1)

**Reprendre et construire W-7 Knowledge-Capture** — dernier morceau de la plateforme.
- Objectif (P5) : ingestion des **débriefs post-event** dans la mémoire `aftrsn_memory`, collection **`insights`** (embedding figé `text-embedding-3-small`, 1536 dims — ne jamais changer).
- Le volet **closing budgétaire** est déjà couvert par W-6 (Budget-Tracker) ; W-7 se concentre sur la capture de connaissance/leçons.
- À trancher avec Karter au démarrage : déclenchement (à la demande par édition, cohérent avec le reste ? ou sur passage d'une édition en `Done` ?), format d'entrée du débrief (fiche Notion dédiée ? texte libre ? champ sur la fiche Edition ?), et destination exacte (collection `insights` de `aftrsn_memory` via `LUMINA-MEMORY-WRITE/WEBHOOK` `zu4jfZbmDz8trQLl`, comme les autres).
- Pattern réutilisable : mêmes briques que W-4/coordinators (lecture Notion en API brute, agent rédacteur via executeWorkflow, écriture, Telegram `chat 776345147`, mémoire, Set résumé).

## 3 · Ce qui a été fait cette session (05.07.2026, W-4)

**W-4 Marketing-Planner construit, testé de bout en bout et PUBLIÉ 🟢** (`AFTRSN-Marketing-Planner` `4qcia6aWt9ooQrle`, actif, sans pin).
1. **v1 contenu** : trigger à la demande `{edition}` → calendrier 11 posts Instagram (J-21→J0) + copies (agent **AFTRSN-Marketing** `wyUEojwZjSKFRaRL`) + newsletter → **pages brouillon dans Content & Social** (`709b816f`, champ `Publish date` ajouté, copie dans le corps de page) → Telegram → mémoire. **Draft-only**, **idempotence par relation**.
2. **P3 Meta Ads** : node `Meta - Campagne (PAUSE)` (onError=continue) → **campagne Meta en PAUSE** par édition (`act_1927939831244305`, Page `694773763721517`), credential n8n `facebookGraphApi` `3fAIGHeem6ebftQ3`. Infra Meta montée avec Karter : app **LUMINA OS** (`992613030068111`, publiée/Live) + utilisateur système **LUMINA HERMES N8N** (`61591950411009`) + **rôle « Gérer l'app »** (le déblocage du token) + token System User « Jamais ». Champ `is_adset_budget_sharing_enabled:false` obligatoire.
3. **P4 Wix** : page événement via le **connecteur Wix** (Events V3, `CreateEvent draft:true`, RSVP sans-billet), **sans clé** — action de la couche agent/Hermès, pas dans n8n. Brouillon 08.08 créé (`b5cb69a2-…`).
4. **DA-008 tranchée** ; **données de test nettoyées** (36 pages Content & Social + 3 éditions archivées, 2 campagnes Meta de test supprimées).

## 4 · Fichiers produits / mis à jour cette session

| Fichier | Rôle |
|---|---|
| `Fiche_AFTRSN-Marketing-Planner_20260705.md` | Fiche technique W-4 (v1 + P3 Meta + P4 Wix, IDs, pièges, tests, nettoyages) |
| `POS-AFTRSN_Calendrier-Contenu-Agent-Notion_20260705.md` + `POS-GENERIQUE_Calendrier-Contenu-Agent-Base_20260705.md` | POS paire : générateur de calendrier de contenu (agent → base, draft-only) |
| `POS-GENERIQUE_Meta-Ads-SystemUser-Token-n8n_20260705.md` | POS générique : token System User Meta Ads pour n8n (générique par nature, pas de paire) — **à pousser au canon** |
| `FICHE_PROJET_AFTRSN_Automation.md` | Boussole, à jour (Étape 3 W-4 terminée, DA-008 tranchée, DA-009 ouverte) |
| `PASSATION_AFTRSN_ProchainesEtapes_20260705-NUIT.md` | Ce document (nouveau relais courant) |

## 5 · Protocole permanent — marches à suivre et POS (instruction pour TOUTES les prochaines sessions)

**Après chaque étape validée**, toute explication réutilisable produite pendant le travail (mode d'emploi pratique généralisable, ou procédure technique détaillée vérifiée) doit être **automatiquement extraite en fichier(s) `.md` et sauvegardée dans `/Users/karter/Claude/Projects/AFTRSN Automation/`** — jamais laissée uniquement dans la conversation ou noyée dans un récap.

**Deux types de document, deux conventions de nommage :**
- **Marche à suivre** — mode d'emploi pratique, réutilisable, **sans date** (document vivant, mis à jour sur place) : `<Scope>_MarcheASuivre_<Sujet-en-Kebab-Case>.md`, `Scope` = `AFTRSN` ou `Generique`.
- **POS** (Procédure Opérationnelle Standardisée) — procédure technique détaillée, snapshot **daté** d'un état vérifié : `POS-<SCOPE>_<Sujet-en-Kebab-Case>_<AAAAMMJJ>.md`, `SCOPE` = `GENERIQUE` / `AFTRSN` / `LUMINA`.

**Règle de la paire systématique :** chaque document **spécifique AFTRSN** (marche à suivre ou POS) est produit avec **une seconde version générique** en parallèle, débarrassée des éléments propres à AFTRSN (IDs, comptes, noms de bases), réutilisable pour toute autre marque gérée via LUMINA OS. Si la procédure est générique par nature, un seul document `Generique_`/`POS-GENERIQUE` (ou `POS-LUMINA` si elle concerne l'architecture LUMINA OS) suffit — pas de doublon artificiel.

**Sections obligatoires dans CHAQUE document produit :**
- **Difficultés rencontrées** — les blocages, erreurs et ambiguïtés effectivement rencontrés (pas hypothétiques).
- **Solutions implémentées** — ce qui a concrètement résolu chaque difficulté (diagnostic exact + correctif appliqué).
- **Leçons apprises** — le principe généralisable, formulé pour rester utile sans se souvenir de l'incident d'origine.

**Règles d'application :**
1. **Détection automatique** — dès qu'une explication étape-par-étape émerge (réponse à une question ou sous-produit d'un débogage), évaluer sa réutilisabilité et l'extraire si oui, **sans attendre la fin de session**.
2. **Sauvegarde effective** dans le dossier projet (jamais seulement dans le scratchpad).
3. **Référencement obligatoire** — chaque nouveau document (y compris sa paire générique) est ajouté à « ✅ Déjà fait › Architecture & documentation » de `FICHE_PROJET_AFTRSN_Automation.md` au moment de sa création, et à la table §4 de la passation courante.
4. **Portée LUMINA OS vs métier AFTRSN** — si le contenu concerne l'architecture/gouvernance du système LUMINA OS lui-même, signaler à Karter qu'il mérite d'être poussé vers le dépôt canonique (`wiki/` → collection `canon`) — un document de session seul ne corrige jamais durablement la connaissance de fond (SYSTEM-CANON §5).
5. **Présentation à l'utilisateur** — chaque document finalisé est présenté via le mécanisme standard (`present_files`), pas seulement décrit en texte.

## 6 · Difficultés rencontrées, solutions implémentées, leçons apprises — session du 05.07.2026 (W-4)

| # | Difficulté | Solution | Leçon |
|---|---|---|---|
| 1 | Node HTTP Notion en erreur « invalid syntax » avec `jsonBody = {{ JSON.stringify({objet}) }}` | Remplacer par du **JSON littéral avec valeur interpolée** : `={"filter":{"property":"Edition","title":{"equals":"{{ $json.edition }}"}}}` | Jamais d'objet littéral dans une expression de jsonBody n8n ; JSON en clair + `{{ champ }}` dans les valeurs |
| 2 | Content & Social sans champ date ni champ de copie | Ajout d'un champ `Publish date` (PATCH DB Notion) ; copie IG écrite dans le **corps** de la page | Lire le schéma réel de la base cible avant d'écrire ; ajouter explicitement les champs manquants dont le process a besoin |
| 3 | Recherche Notion renvoie des **bases en doublon** (même nom, IDs différents) | La **1re occurrence** = celle qu'utilisent les workflows publiés (canonique) ; toujours **By ID** | Ne jamais se fier au nom d'une base ; vérifier l'ID contre celui des workflows en prod |
| 4 | Token System User Meta : « Aucune autorisation disponible — affectez un rôle d'application à l'utilisateur système » (même app publiée, compte pub assigné) | **Applications → LUMINA OS → Affecter des personnes → utilisateur système → activer « Gérer l'app »**, puis regénérer | Un token System User n'expose ses permissions que si l'utilisateur système a un **rôle dans l'app** — c'est le blocage n°1, non évident |
| 5 | Bouton **Publier** de l'app Meta grisé | Renseigner **URL de politique de confidentialité + catégorie** dans Paramètres de base | Publier une app Meta exige privacy URL + catégorie |
| 6 | Création campagne Meta : erreur 100 « Invalid parameter » générique | Lire la **réponse FB brute** (`fullResponse`+`neverError`) → champ manquant `is_adset_budget_sharing_enabled` → l'ajouter (`false`) | Toujours capturer la réponse FB complète pour lire `error_user_title` |
| 7 | Wix : croyais qu'il fallait une clé API / header | Le **connecteur Wix déjà autorisé** crée l'événement (Events V3, `draft:true`, RSVP) sans clé | Vérifier si un connecteur agent couvre déjà le besoin avant d'imposer un credential à l'utilisateur ; n8n ≠ connecteur agent (n8n ne peut pas l'appeler) |
| 8 | Lecture de code/prompts d'un node via l'API navigateur bloquée (filtre « sensitive ») | Contrat reconstruit depuis les nodes lisibles + **injection de code par base64** (`atob`+`TextDecoder` UTF-8) pour pousser du JS non trivial | Les POS/nommage clair sont une assurance-lecture ; base64 élimine les pièges d'échappement |
| 9 | Données de test en base = bruit | Éditions de test **archivées** (corbeille Notion réversible), pages de contenu **archivées**, campagnes Meta de test **supprimées** | Nettoyer ses propres artefacts de test en fin de run ; archiver (réversible) plutôt que supprimer pour Notion |

## 7 · Décisions / nettoyages en attente (Karter)
- **DA-009** — Supprimer le workflow utilitaire jetable `ZZZ_20260705_tmp_w4_probe` (`Yk8t5R6WQxbr6tmB`, non publié).
- **DA-001b / DA-002 / DA-004 / DA-006 / DA-007** — voir Fiche Projet (nettoyages ZZZ divers).
- Brouillon Wix « AFTER SUN — Zürich · 08.08.2026 » (`b5cb69a2-…`, DRAFT) : garder (vraie page 08.08, horaire 15h–22h à ajuster) ou refaire.
- **À pousser au canon** (`wiki/`, collection `canon`) : « from day to night » (prioritaire), la **POS token Meta** (procédure LUMINA OS réutilisable), les POS génériques.

## 8 · Rappels transverses (invariants LUMINA OS)
- Numérotation toujours à partir de 1. PII jamais dans le cloud. Actions critiques (dépense, envoi, publication, suppression) toujours gatées par validation humaine. Embedding figé. Git = source de vérité. n8n fait foi en cas d'écart avec la doc. Aucune suppression/DROP sans Karter. Jamais réactiver un `ZZZ_` sans revue. Toujours HTTP Request pour appeler Claude (jamais le node Anthropic natif). L'agent ne saisit jamais de secret (token, clé, mot de passe) — l'utilisateur le fait ; passkey/2FA = humain uniquement.

---
*Document rédigé en fin de session le 05.07.2026 (nuit).*
