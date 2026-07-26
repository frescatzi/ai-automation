---
type: raw
title: "POS-AFTRSN_Backfill-Outreach-Manuel_20260705"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_backfill-outreach-manuel_20260705.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS AFTRSN — Backfill : intégrer un contact déjà démarché manuellement dans l'outreach automatisé

Procédure vérifiée le 05.07.2026 — intégration de 3 lieux contactés manuellement depuis partners@aftersunpeople.com avant l'automatisation (Maison 25, Soluna, The Palm Three), sans réponse, pour que les relances reprennent automatiquement **dans les fils d'origine**. Version générique : `POS-GENERIQUE_Backfill-Outreach-Manuel_20260705.md`. Skill Hermès correspondant : n° 990.

## 1. Prérequis

- Processus `AFTRSN-Venue-Coordinator` (ou Artist) publié, piloté par la base Venues/DJs-Artists 🎭 (statut `To contact`, champs `Email`, `1st Email`, `Follow-up 1/2`, `Gmail Thread`).
- Utilitaire `AFTRSN-UTIL-Partners-Sent-Lookup` (`ZsFiSf7pc5RQAKyc`, dossier `AFTRSN-01-INFRASTRUCTURE`) : manuel, credential `PARTNERS-GMAIL`, node Gmail getAll avec requête `q` à adapter.
- Depuis la v1.2.1, la seconde relance part 4 jours après la première (et non 7 jours après le premier email) — indispensable pour le backfill (voir §3.4).

## 2. Procédure

1. **Collecter les noms** des cibles déjà contactées auprès de Karter (les noms d'usage oraux peuvent différer des noms réels — voir §3.3).
2. **Retrouver les envois** : adapter la requête de l'utilitaire (`in:sent (<nom1> OR <nom2> …)`), exécuter, lire dans la sortie : destinataire (`To`), date (`internalDate`), **`threadId`**, sujet. Si 0 résultat : variantes de graphie (collée/séparée, domaine du site), puis en dernier recours `in:sent` seul (30 derniers) et filtrage à l'œil.
3. **Consigner dans la base** (fiche existante ou à créer) : `Email` = destinataire réel, `1st Email` = date réelle du premier envoi, `Gmail Thread` = threadId, `Notes` = mention « contacté manuellement le JJ.MM, sans réponse » + sujet d'origine, `Status` = **`To contact`**.
4. **Laisser faire le processus** : au run suivant, il détecte « premier email ancien, pas de Follow-up 1 » → brouillon de **relance 1 dans le fil d'origine** ; puis relance 2 quatre jours plus tard si toujours rien.

## 3. Difficultés rencontrées

1. La recherche Gmail par mots-clés a raté 2 cibles sur 3 : « The Palm Tree » s'appelait en réalité **The Palm Three**, et « Studio 45 » était en réalité **Maison 25** (contact via une agence tierce, info@urbanagency.ch).
2. Les graphies collées ne matchent pas les mots séparés dans Gmail (« Studio45 » ≠ « studio 45 »).
3. Les fiches backfillées au premier email ancien auraient reçu **deux relances à un jour d'intervalle** (relance 1 lundi, relance 2 mardi) : la condition d'origine calculait la seconde relance sur la date du premier email (J+7), déjà largement dépassée.
4. Clics « Execute workflow » perdus à répétition entre les rechargements de page (références UI périmées).

## 4. Solutions implémentées

1. Repli sur `in:sent` sans filtre + lecture des 30 derniers envoyés : les vrais noms et destinataires sont apparus immédiatement ; confirmation par Karter en cours de route (il a reconnu les adresses).
2. Requêtes multi-variantes (nom exact, collé, séparé, domaine) avant le repli.
3. Correctif publié (v1.2.1 lieux / v1.0.2 artistes) : relance 2 déclenchée par « `Follow-up 1` ≥ 4 jours », plus par « `1st Email` ≥ 7 jours ».
4. Re-screenshot avant chaque clic + vérification qu'un **nouveau** run existe dans `/rest/executions` avant d'en attendre le résultat.

## 5. Leçons apprises

1. **Le nom d'usage n'est pas une clé fiable** : la source de vérité d'un historique de contact est la boîte d'envoi elle-même — quand la recherche par nom échoue, lister et regarder.
2. Un backfill est un **test de bord du processus** : il a révélé un défaut de cadence (relances consécutives) invisible sur le chemin nominal. Intégrer des données anciennes dans un processus neuf mérite une relecture des conditions temporelles.
3. Les intervalles d'une séquence se calculent **sur l'événement précédent de la séquence**, pas sur son origine — règle générale pour toute chaîne de relances.
4. Conserver les fils d'origine (threadId) vaut le détour : une relance dans le fil garde tout le contexte côté destinataire et évite de repartir de zéro.
