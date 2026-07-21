---
type: wiki
title: SOP — Répondeur email à brouillons via agent (pattern draft-only)
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-07-21--pos-generique-repondeur-email-drafts-agent-20260704.md
related:
  - wiki/sop/sop-calendrier-contenu-agent.md
  - wiki/concept-memoire-vivante-agents.md
  - wiki/concept-n8n-credentials.md
updated: 2026-07-21
---

# SOP — Répondeur email à brouillons via agent (pattern draft-only)

Patron réutilisable pour toute marque/projet géré via LUMINA OS : une boîte email surveillée, un agent qui rédige des brouillons, un humain qui garde la main sur l'envoi. Vérifié le 2026-07-04.

## Principe et gouvernance

L'envoi d'un email est une **action critique** : toujours gatée par l'humain, quel que soit le niveau de confiance dans l'agent. Le workflow se limite à lire → rédiger un brouillon → déposer le brouillon dans le fil → notifier. L'humain relit et envoie depuis son client email. Même logique de gate que [[sop/sop-calendrier-contenu-agent]] (draft-only, aucune publication/envoi automatique).

## Chaîne type (10 nodes)

1. **Schedule** — fréquence adaptée au volume ; l'horaire est un bon défaut.
2. **Lecture inbox** — non-lus récents (`is:unread newer_than:2d`), exclusions dans la requête (`-from:no-reply -category:promotions/social/updates`), limite (ex. 20).
3. **Filtre (Code)** — exclusions par expéditeur (regex `no-?reply|newsletter|notification|mailer-daemon`), extraction expéditeur/objet/corps, **rejet des corps vides** (on ne répond pas à rien).
4. **Agent rédacteur** (Execute Workflow vers l'agent de la marque) — sortie balisée en un seul appel : `RESUME_EMAIL:` (1-2 phrases), `RESUME_DRAFT:` (1-2 phrases), `---DRAFT---` (corps complet). Règles du prompt : langue de l'email reçu, ton de la marque, contexte auto-détecté, questions plutôt qu'inventions, **jamais d'engagement ferme/dépense/contrat**, signature de la marque.
5. **Découpage (Code)** — parse les 3 parties, repli sur extraits tronqués si les balises manquent.
6. **Création du brouillon** en réponse dans le fil (threadId + destinataire = expéditeur).
7. **Idempotence** — `markAsRead` (simple) ou label dédié (plus robuste si l'humain lit sa boîte en parallèle).
8. **Notification** (Telegram/Slack…) — format humain sans jargon : titre court, expéditeur, objet, résumé du reçu, résumé du draft.
9. **Journalisation mémoire** — un épisode par run utile (voir [[concept-memoire-vivante-agents]], primitive WRITE).
10. **Résumé** `{status, drafts}`.

Comportement à vide : 0 email → arrêt silencieux après la lecture (pas de notification vide, pas d'épisode) — comportement souhaitable pour un run fréquent.

## Difficultés & solutions

| Difficulté | Solution |
|---|---|
| `403 Forbidden` sur l'API email malgré une credential valide | Activer l'API dans la console cloud du fournisseur — prérequis d'infrastructure, pas un bug de workflow. |
| Parseur MIME écrit contre un format supposé → 100 % des emails filtrés à tort | Inspecter la sortie réelle d'une exécution, puis réécrire le parseur contre le format observé. |
| Email de test sans corps rejeté, diagnostic initialement orienté vers le mauvais suspect (le filtre) | Diagnostiquer par les données (longueur du corps) avant de toucher au code ; retester avec un contenu réaliste. |
| Correctif API écrasé par l'autosave d'un éditeur resté ouvert | Recharger systématiquement l'éditeur après toute écriture API. |
| Identifiant de chat de notification introuvable (les API de bots n'acceptent pas les alias en privé) | Demander à l'utilisateur d'écrire à un bot d'introspection (ex. `@rawdatabot` sur Telegram) qui renvoie son propre id. |

## Leçons apprises

1. **Prérequis d'infrastructure d'abord** : une API désactivée côté fournisseur mime un bug de workflow — vérifier l'activation avant de déboguer.
2. **Coder contre le format observé, jamais supposé** : une exécution d'inspection coûte 30 secondes, un parseur faux coûte une session.
3. **Un seul appel d'agent, sortie balisée** : résumés + brouillon en une passe, découpés ensuite — moins cher, plus cohérent, parsable avec repli.
4. **Le format des notifications est une exigence produit** : nom métier du processus, pas de codes techniques, contenu utile à la décision (qui, quoi, résumé reçu, résumé proposé).
5. **Tester le chemin vide et le chemin plein** : boîte vide, email vide, email réel — trois comportements distincts à valider.

## Voir aussi

- [[sop/sop-calendrier-contenu-agent]] — autre patron draft-only : génération de contenu en base, publication toujours humaine.
- [[concept-memoire-vivante-agents]] — primitive WRITE utilisée à l'étape 9 (journalisation mémoire).
- [[concept-n8n-credentials]] — piège « API désactivée côté fournisseur mime une credential invalide » à rapprocher des pannes de credentials.
