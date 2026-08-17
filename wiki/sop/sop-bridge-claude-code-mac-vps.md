---
type: wiki
title: "SOP — Bridge Claude Code Mac↔VPS via tunnel Cloudflare (Operator OS)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-exact-bridge-claude-code.md
updated: 2026-08-17
related: ["concept-hermes-agent", "synthese-lumina-ai-os", "sop-dream-engine-cron-prescriptions-llm", "concept-n8n-credentials"]
---

# SOP — Bridge Claude Code Mac↔VPS via tunnel Cloudflare (Operator OS)

**Idée centrale :** patron pour permettre à un service serveur (VPS, derrière NAT dans l'autre sens — c'est lui qui doit *joindre* la machine locale) d'interroger le **Claude Code CLI** installé sur une machine locale (Mac) qui n'a pas d'IP publique. La machine locale ouvre la connexion **sortante** (tunnel), le serveur appelle l'URL publique exposée, et un header d'authentification statique protège chaque requête. Implémentation de référence : **Operator OS** (LUMINA), Hermes (VPS) → Claude Code (Mac) — ✅ verrouillé, validé live au 2026-08-17.

## 1. Architecture du pont

- **Serveur appelant** (VPS, Operator OS/FastAPI) : ne peut pas joindre le Mac directement (NAT).
- **Machine locale** (Mac, Claude Code CLI installé) : héberge un petit serveur HTTP stdlib qui exécute `claude -p "<prompt>"` et renvoie la réponse en JSON.
- **Tunnel** : `cloudflared tunnel --url http://127.0.0.1:<port>` — quick tunnel sans compte, expose une URL publique `https://<random>.trycloudflare.com`. Aucune garantie d'uptime, **URL éphémère qui change à chaque relance**.
- **Auth** : header statique (ex. `X-Bridge-Key`) vérifié par le serveur local sur chaque requête entrante — sans lui, quiconque connaît l'URL publique peut interroger le CLI.
- Le serveur appelant garde l'URL du tunnel + la clé dans sa config (ex. `claude_bridge_url` / `claude_bridge_key`) et doit la mettre à jour à chaque relance du tunnel.

## 2. Marche à suivre

1. Vérifier qu'on est bien sur la machine locale avant toute commande (confusion fréquente terminal local ↔ SSH serveur — vérifier le prompt/hostname).
2. Libérer le port du bridge s'il est déjà occupé : `lsof -ti :<port> | xargs kill -9`.
3. Démarrer le bridge avec la clé d'auth en variable d'environnement, puis vérifier qu'elle est bien celle attendue avant de continuer.
4. Lancer `cloudflared tunnel --url http://127.0.0.1:<port>` et récupérer la nouvelle URL publique.
5. Mettre à jour la config du serveur appelant (URL + clé) pour qu'il pointe vers le tunnel actuel.

## 3. Pièges rencontrés & correctifs

| Erreur observée | Cause | Solution |
|---|---|---|
| Commandes sans effet / mauvais résultat | terminal local et session SSH serveur confondus | vérifier le hostname du prompt avant toute commande |
| `OSError: [Errno 48] Address already in use` | deux instances du bridge tournaient sur le même port | `lsof -ti :<port> \| xargs kill -9` avant de relancer |
| `401 Unauthorized` | clé du serveur local ≠ clé envoyée par l'appelant | resynchroniser la clé des deux côtés, vérifier par `echo` avant de relancer |
| `HTTP Error 530` | le tunnel cloudflared s'était arrêté pendant le redémarrage du bridge | relancer le tunnel et remettre à jour la config appelant avec la nouvelle URL |
| CLI absent côté serveur | tentative d'exécuter le bridge sur le serveur au lieu de la machine locale | le bridge doit tourner uniquement là où le CLI est installé |

## 4. Leçons clés

- Un serveur qui « ne répond pas » dans un terminal est souvent un serveur qui **attend** — un terminal bloqué sans prompt = signe que le process tourne, pas qu'il est planté.
- Un quick tunnel Cloudflare (sans compte) n'a **aucune garantie d'uptime** et change d'URL à chaque relance ; pour un besoin stable, passer à un tunnel nommé (compte Cloudflare).
- Toujours exiger un header d'authentification statique sur le serveur exposé par tunnel : sans clé, l'URL publique seule suffit à interroger le CLI.

## Voir aussi

- [[concept/concept-hermes-agent]] — Hermes (VPS) est l'appelant de ce pont dans l'implémentation Operator OS de référence.
- [[synthese/synthese-lumina-ai-os]] — place d'Operator OS dans l'écosystème LUMINA plus large.
- [[sop/sop-dream-engine-cron-prescriptions-llm]] — autre composant Operator OS interrogeant Claude, sur la même infra VPS.
- [[concept/concept-n8n-credentials]] — patron général de gestion des clés/secrets statiques (ne jamais coder en dur, toujours dans un credential/variable d'environnement).
