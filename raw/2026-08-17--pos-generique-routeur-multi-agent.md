---
type: raw
title: "POS-GENERIQUE-routeur-multi-agent"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique-routeur-multi-agent.md"
captured: 2026-08-17
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE : router plusieurs LLM derrière une seule interface

Date : 2026-08-17 · Portée : générique, tout dashboard voulant interroger plusieurs LLM depuis une UI unique, aucun identifiant réel.

## Principe

Pour dialoguer avec plusieurs LLM (modèles internes, API cloud, CLI local) depuis une seule UI, on définit un registre d'agents où chaque agent porte un `type` qui détermine son backend. Le routeur lit le type et aiguille vers le bon canal. L'UI se contente d'afficher les agents et d'envoyer `{agent_id, prompt}`.

## Marche à suivre

1. Définir un registre d'agents (fichier de config) : `id`, `name`, `type`, `model`, `description`, `status`.
2. Implémenter un backend par type : API interne, API cloud via webhook, CLI local via bridge/tunnel.
3. Exposer deux routes : `GET /agents` (liste) et `POST /agents/chat` (prompt + agent_id).
4. Dans l'UI, afficher une carte par agent avec un indicateur de statut, et une zone de chat.
5. Router selon le type et renvoyer la réponse normalisée.

## Garde-fous

1. Ne jamais exposer un secret dans l'UI : le backend gère les clés.
2. Tester chaque backend en CLI avant de brancher l'UI.
3. Normaliser la réponse (champ `response` unique) quel que soit le backend.
4. Garder un statut par agent (online/offline) pour signaler les backends indisponibles.

## Difficultés rencontrées

1. Un backend répond un format brut (tableau JSON) différent du format attendu.
2. Un middleware d'auth bloque les appels du frontend.
3. Un crédit API épuisé rend un backend silencieux.

## Solutions implémentées

1. Parser le format brut dans la fonction du backend (extraire le texte des `content[]`).
2. Autoriser les routes statiques sans auth et envoyer l'en-tête d'auth depuis le frontend.
3. Afficher le statut dégradé et documenter la dépendance crédit.

## Lessons learned

1. L'API d'un LLM génère du texte, elle ne lit pas une mémoire : séparer génération et mémoire durable.
2. Isoler chaque backend derrière une fonction dédiée facilite le débogage.
3. Le statut live d'un agent (vert/rouge) est plus parlant qu'un message d'erreur après coup.