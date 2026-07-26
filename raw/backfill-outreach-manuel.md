---
type: wiki
title: "AFTRSN — Backfill : intégrer un contact démarché manuellement dans l'outreach automatisé"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [AFTRSN/raw/2026-07-21--pos-aftrsn-backfill-outreach-manuel-20260705.md]
related: [[architecture-processus-metier]], [[methode-construction-workflow-n8n]]
updated: 2026-07-21
---

# AFTRSN — Backfill : intégrer un contact démarché manuellement dans l'outreach automatisé

Procédure vérifiée le 05.07.2026 (3 lieux — Maison 25, Soluna, The Palm Three — contactés manuellement depuis `partners@aftersunpeople.com` avant l'automatisation, sans réponse) pour faire reprendre les relances automatiques du processus **Coordination lieux** (voir [[architecture-processus-metier]] §Processus transverses) **dans le fil Gmail d'origine**, sans dupliquer le premier contact.

## Prérequis

- Le processus concerné (`AFTRSN-Venue-Coordinator` ou `-Artist`) est publié et piloté par la base Venues/DJs-Artists 🎭 (statut `To contact`, champs `Email`, `1st Email`, `Follow-up 1/2`, `Gmail Thread`).
- Utilitaire manuel `AFTRSN-UTIL-Partners-Sent-Lookup` (`AFTRSN-01-INFRASTRUCTURE`, credential `PARTNERS-GMAIL`) : recherche Gmail `getAll` sur `in:sent`, requête `q` à adapter par cas.
- Depuis la v1.2.1 (lieux) / v1.0.2 (artistes), la relance 2 se déclenche sur `Follow-up 1 ≥ 4 jours`, plus sur `1er email ≥ 7 jours` — condition indispensable pour que le backfill ne parte pas avec deux relances rapprochées (voir Leçons).

## Procédure

1. Collecter auprès du responsable partenariats les noms des cibles déjà démarchées — les noms d'usage oraux ne sont pas fiables (voir Leçons).
2. Retrouver l'envoi réel via l'utilitaire (`in:sent (nom1 OR nom2 …)`) : destinataire, date, **`threadId`**, sujet. En cas de 0 résultat, tester les variantes de graphie (collée/séparée, domaine), puis en dernier recours parcourir `in:sent` seul (30 derniers) à l'œil.
3. Consigner dans la fiche Notion (existante ou créée) : `Email` = destinataire réel, `1st Email` = date réelle du premier envoi, `Gmail Thread` = threadId, `Notes` = « contacté manuellement le JJ.MM, sans réponse » + sujet d'origine, `Status` = `To contact`.
4. Laisser le processus détecter au run suivant « premier email ancien, pas de Follow-up 1 » : il rédige la relance 1 **dans le fil d'origine**, puis la relance 2 selon la cadence corrigée.

## Leçons apprises

- **Le nom d'usage n'est pas une clé de recherche fiable** (« The Palm Tree » → en réalité *The Palm Three* ; « Studio 45 » → en réalité *Maison 25*, contactée via une agence tierce). Quand la recherche par nom échoue, lister les derniers envois et vérifier à l'œil plutôt que de multiplier les variantes à l'infini — la boîte d'envoi est la source de vérité d'un historique de contact, pas la mémoire humaine du nom.
- Les graphies collées ne matchent pas les mots séparés dans Gmail (`Studio45` ≠ `studio 45`) : tester les deux avant de conclure à un 0 résultat.
- **Un backfill est un test de bord du processus** : injecter une donnée ancienne (premier email vieux de plusieurs semaines) dans une séquence de relances neuve a révélé un défaut de cadence invisible sur le chemin nominal (deux relances à un jour d'intervalle, calculées à tort depuis l'origine de la séquence). Règle générale pour toute chaîne de relances : les intervalles se calculent **sur l'événement précédent de la séquence**, jamais sur son origine.
- Conserver et réutiliser le `threadId` d'origine vaut le détour : la relance garde tout le contexte côté destinataire et évite de repartir de zéro.
- Fiabilité UI n8n pendant l'opération : re-vérifier par capture d'écran avant chaque clic « Execute workflow » et confirmer l'apparition d'un **nouveau** run dans `/rest/executions` avant d'attendre un résultat (références DOM périmées après rechargement — voir aussi [[methode-construction-workflow-n8n]] §Refactoring et API REST interne n8n).
