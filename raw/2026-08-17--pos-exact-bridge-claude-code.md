---
type: raw
title: "POS-EXACT-bridge-claude-code"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact-bridge-claude-code.md"
captured: 2026-08-17
vault: ai-automation
brand: null
immutable: true
---

# POS-EXACT - Bridge Claude Code Mac↔VPS (Operator OS)

**Date :** 2026-08-17 · **Projet :** Operator OS (LUMINA) · **Objet :** verrouiller le pont qui permet à Hermes (VPS) d'interroger Claude Code CLI (Mac) via un tunnel Cloudflare.
**Statut :** ✅ VERROUILLÉ & VALIDÉ LIVE.

---

## 1. Architecture du pont

- **VPS** (`root@aftersun-n8n`, IP publique 178.105.197.189) : Operator OS (FastAPI, port 8090) est l'appelant.
- **Mac** (`karter@Enso`) : Claude Code v2.1.220 installé en local, Python 3.9 (CommandLineTools).
- **Bridge** : script `claude_bridge_mac.py` sur le Mac, serveur HTTP stdlib sur `127.0.0.1:8000`, exécute `claude -p "<prompt>"` et renvoie la réponse JSON.
- **Tunnel** : `cloudflared tunnel --url http://127.0.0.1:8000` expose une URL publique `https://<random>.trycloudflare.com` (quick tunnel account-less, sans garantie d'uptime).

## 2. Config exacte

| Élément | Valeur |
|---|---|
| Fichier bridge (VPS, source) | `/opt/data/operator-os/claude-bridge-mac/claude_bridge_mac.py` |
| Fichier bridge (Mac) | `~/claude_bridge_mac.py` |
| Port | 8000 |
| Clé | `p3ymt-YgPZ8-NXpzE-iDjWk-6DcR5` (définie par Karter, header `X-Bridge-Key`) |
| Tunnel actuel | `https://compilation-warming-editor-product.trycloudflare.com` (éphémère, change à chaque relance) |
| Config Operator OS | `config.json` → `claude_bridge_url` + `claude_bridge_key` |

**Principe :** le VPS ne peut pas joindre le Mac (NAT). Le Mac ouvre la connexion sortante (cloudflared), le VPS appelle l'URL publique, et le header `X-Bridge-Key` authentifie chaque requête.

---

## Difficultés / Solutions / Lessons learned

### Difficultés
- Confusion récurrente entre le Terminal Mac et le SSH VPS : les commandes étaient lancées dans `root@aftersun-n8n` au lieu du Mac.
- `OSError: [Errno 48] Address already in use` : deux instances du bridge tournaient sur le port 8000.
- `401 Unauthorized` : la clé du serveur Mac ne correspondait pas à celle envoyée par le VPS.
- `HTTP Error 530` : le tunnel cloudflared s'était arrêté pendant le redémarrage du bridge.
- `claude` absent du VPS : le bridge doit tourner sur le Mac, jamais sur le serveur.

### Solutions
- Vérifier le hostname du prompt avant toute commande (`karter@Enso` = Mac, `root@aftersun-n8n` = VPS).
- `lsof -ti :8000 | xargs kill -9` pour libérer le port avant de relancer.
- Relancer le bridge avec `export CLAUDE_BRIDGE_KEY="..."` puis `echo` de vérification de la clé.
- Relancer `cloudflared tunnel --url http://127.0.0.1:8000` et mettre à jour `config.json` avec la nouvelle URL.

### Lessons learned
- Un serveur ne "répond" pas : il attend. Un terminal bloqué sans prompt = serveur en marche.
- Le quick tunnel Cloudflare n'a aucune garantie d'uptime et change d'URL à chaque relance. Pour du stable, passer à un tunnel nommé.
- Toujours exiger une clé (header `X-Bridge-Key`) : sans clé, n'importe qui connaissant l'URL peut interroger Claude Code.

---

*POS-EXACT bridge-claude-code - 2026-08-17, verrouillé. Le script claude_bridge_mac.py fait foi.*