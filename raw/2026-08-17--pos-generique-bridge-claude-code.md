---
type: raw
title: "POS-GENERIQUE-bridge-claude-code"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique-bridge-claude-code.md"
captured: 2026-08-17
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE : relier un agent serveur à un CLI IA local via tunnel

Date : 2026-08-17 · Portée : générique, tout couple serveur distant / machine locale hébergeant un CLI IA, aucun identifiant réel.

## Principe

Un serveur distant ne peut pas joindre une machine derrière NAT. Pour interroger un CLI IA installé localement (Claude Code, Codex, etc.) depuis le serveur, on inverse la connexion : la machine locale expose le CLI via un petit serveur HTTP, et un tunnel (Cloudflare quick) rend ce serveur joignable publiquement. Le serveur distant appelle l'URL publique avec un header de clé pour s'authentifier.

## Marche à suivre

1. Sur la machine locale, écrire un mini serveur HTTP (Python stdlib) qui reçoit un POST `{"prompt": "..."}`, exécute `<cli> -p "<prompt>"`, et renvoie `{"response": "..."}`.
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

## Difficultés rencontrées

1. Port déjà occupé par une instance précédente du serveur.
2. Clé divergente entre le serveur local et le client distant (401).
3. Tunnel coupé pendant un redémarrage (erreur 530).

## Solutions implémentées

1. `lsof -ti :PORT | xargs kill -9` pour libérer le port.
2. Exporter la clé puis la vérifier avec `echo` avant de lancer le serveur.
3. Relancer cloudflared et mettre à jour l'URL côté client.

## Lessons learned

1. Un serveur qui attend ne rend pas la main : un terminal "bloqué" est souvent un serveur en marche.
2. Les tunnels sans compte n'ont aucune garantie d'uptime ; pour du stable, préférer un tunnel nommé.
3. La clé d'authentification est indispensable dès que l'URL devient publique.