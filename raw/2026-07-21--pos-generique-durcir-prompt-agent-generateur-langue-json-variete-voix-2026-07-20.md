---
type: raw
title: "POS-GENERIQUE_durcir-prompt-agent-generateur-langue-json-variete-voix_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_durcir-prompt-agent-generateur-langue-json-variete-voix_2026-07-20.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Durcir le prompt d'un agent générateur (langue, JSON strict, variété, voix)

Objet : recette réutilisable pour fiabiliser un agent LLM qui produit un livrable structuré consommé par un runner (ex. Iris → M2).

## Symptômes typiques
- L'agent ajoute un préambule ou du commentaire avant/après le JSON (fragilise le parsing aval).
- Il bascule de langue (ex. « default French » qui fuit).
- Les champs d'un même livrable se ressemblent trop (cohérence poussée jusqu'à la monotonie).
- Ponctuation non conforme (tirets cadratins).

## Leviers dans le systemMessage (par ordre d'impact)
1. Contrat de sortie explicite : « quand un format est demandé (JSON à clés nommées), ne renvoyer QUE cet objet, sans préambule, commentaire, explication ni code fence, langue par défaut X, tous les champs présents et non vides. » Retirer toute consigne qui invite à ajouter des options/variantes hors format.
2. Langue épinglée (par défaut + gouvernée par le brief), jamais « langue de l'utilisateur » si la sortie a un standard.
3. Voix embarquée en dur (ceinture) EN PLUS de la mémoire/RAG (bretelles) : ne pas déléguer 100 % à un tool qui peut renvoyer vide.
4. Consigne de variété : renouveler l'angle entre exécutions ; interdire les champs quasi-identiques dans un même livrable.
5. Ponctuation : bannir explicitement le tiret cadratin (règle Karter), écrire « comme un éditeur humain ».

## Méthode de validation (sans publier)
- Capturer une sortie AVANT sur un brief réel (via le chat trigger de l'agent, aucune écriture aval).
- Patcher le prompt (n8n_update_partial_workflow, updateNode sur `parameters.options.systemMessage`).
- Rejouer le MÊME brief → comparer.
- Ajouter un cas « piège » (données ambiguës) pour prouver que le contrat de sortie tient là où l'ancien cassait.
- Garder le garde-fou aval comme filet.

## Pièges
- L'agent lit la version ACTIVE : vérifier que le patch est bien pris (le run rejoue le system prompt dans inputOverride, s'y fier).
- Un tool mémoire qui renvoie vide fait retomber la voix en générique : peupler la mémoire d'abord, ou embarquer la voix.
