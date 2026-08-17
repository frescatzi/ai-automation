---
type: wiki
title: "SOP générique — Relier un agent serveur à un CLI IA local via tunnel"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-08-17--pos-generique-bridge-claude-code.md
updated: 2026-08-17
related: ["sop-bridge-claude-code-mac-vps", "concept-n8n-credentials", "sop-registre-agents-multi-backends"]
---

# SOP générique — Relier un agent serveur à un CLI IA local via tunnel

**Idée centrale :** patron réutilisable, hors marque, pour tout couple **serveur distant / machine locale** où le serveur doit interroger un **CLI IA installé localement** (Claude Code, Codex, etc.) que la machine locale héberge derrière NAT (donc injoignable directement). On inverse la connexion : la machine locale expose le CLI via un petit serveur HTTP, un tunnel (Cloudflare quick) rend ce serveur joignable publiquement, et le serveur distant appelle l'URL publique avec un header de clé pour s'authentifier. Implémentation concrète connue : [[sop-bridge-claude-code-mac-vps]] (Operator OS / Hermes ↔ Claude Code Mac).

## Marche à suivre

1. Sur la machine locale, écrire un mini serveur HTTP (stdlib) qui reçoit un POST `{"prompt": "..."}`, exécute `<cli> -p "<prompt>"`, et renvoie `{"response": "..."}`.
2. Protéger le serveur avec une clé : exiger un header (ex. `X-Bridge-Key`) égal à une valeur d'environnement.
3. Lancer le serveur sur `127.0.0.1:8000`.
4. Dans un second terminal, exposer le port via `cloudflared tunnel --url http://127.0.0.1:8000`.
5. Récupérer l'URL publique affichée par cloudflared.
6. Configurer le serveur distant pour appeler cette URL avec la clé dans le header.

## Garde-fous

1. Ne jamais lancer le bridge sur le serveur : il doit tourner sur la machine où le CLI est installé.
2. Ne jamais exposer le port sans clé d'authentification.
3. Le quick tunnel change d'URL à chaque relance : mettre à jour la config distante à chaque fois.
4. Vérifier le hostname du terminal avant de lancer (local vs SSH distant).

## Difficultés rencontrées & solutions

| Erreur observée | Cause | Solution |
|---|---|---|
| Port déjà occupé | instance précédente du serveur toujours active | `lsof -ti :PORT \| xargs kill -9` pour libérer le port |
| `401` | clé divergente entre le serveur local et le client distant | exporter la clé puis la vérifier avec `echo` avant de lancer le serveur |
| `530` | tunnel coupé pendant un redémarrage | relancer cloudflared et mettre à jour l'URL côté client |

## Leçons clés

- Un serveur qui attend ne rend pas la main : un terminal « bloqué » est souvent un serveur en marche.
- Les tunnels sans compte n'ont aucune garantie d'uptime ; pour du stable, préférer un tunnel nommé.
- La clé d'authentification est indispensable dès que l'URL devient publique.

## Voir aussi

- [[sop-bridge-claude-code-mac-vps]] — implémentation concrète de ce patron : Hermes (VPS) ↔ Claude Code (Mac) dans Operator OS/LUMINA, avec pièges et correctifs observés en usage réel.
- [[concept-n8n-credentials]] — patron général de gestion des clés/secrets statiques (ne jamais coder en dur, toujours en variable d'environnement/credential).
- [[sop-registre-agents-multi-backends]] — ce bridge est l'implémentation type du backend `CLI local` dans un registre d'agents multi-backends.
