---
type: raw
title: "LUMINA — POS GENERIQUE — Runbook systeme multi-agents hub-and-spoke avec memoire centrale vectorielle"
source_url: "drive:1cL6CrxNSS91vh6meqiUvokc2IaGfcar4"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Runbook : système multi-agents hub-and-spoke + mémoire centrale vectorielle

**Procédure Opérationnelle Standardisée réutilisable** (toute marque / tout projet « AI OS »).
Architecture cible : 1 Superviseur orchestrant N spécialistes, chacun lisant une **mémoire centrale** (vector store) via un outil/MCP, avec validation humaine sur le critique.

> Checklist concise et indépendante de l'implémentation. Numérotation à partir de 1.

---

## 1. Principes de nommage & frontières

- **Infra partagée** (mémoire, ingestion, MCP, base vectorielle) = un préfixe neutre (ex. `LUMINA-`).
- **Items spécifiques à une marque** (les agents d'une marque) = préfixe de la marque (ex. `AFTRSN-`).
- L'**architecture** superviseur+spécialistes est un patron réutilisable ; chaque **instance** est rattachée à une marque et tape la **même** mémoire/MCP partagés.

## 2. Santé de la mémoire (vérif périodique)

1. Appeler le point d'entrée (MCP/webhook) avec une question de contrôle connue.
2. Attendu : résultats pertinents avec scores. Sinon → diagnostic §3.
3. Surveiller le **taux d'échec** des exécutions et la **fraîcheur** de l'ingestion.

## 3. Mémoire en panne — triage

1. Logs/Executions → identifier le **workflow réellement en échec** (le serveur MCP n'est qu'un relais).
2. Lire l'erreur du node fautif.
3. Causes par ordre de probabilité : **credential morte** → **table vide** (ingestion cassée / TRUNCATE interrompu) → **SQL fragile** → seuil/dimension → permissions.
4. Corriger la cause amont, **repeupler** si la table est vide, **re-tester end-to-end**.

## 4. Règles d'or d'implémentation

- **Requêtes paramétrées** pour toute écriture en base (`$1..$n` + tableau de valeurs) — jamais de concaténation avec du contenu riche.
- **Idempotence** de l'ingestion (`ON CONFLICT … DO NOTHING`, hash de contenu) ; attention au pattern **TRUNCATE-puis-reload** (échec post-truncate = table vide).
- **Une seule credential par service**, réutilisée ; en cas de rotation, **réassigner partout**.
- **Vecteur** passé en littéral texte `[..]` casté `::vector` via paramètre (sans quotes manuelles).

## 5. Câblage standard d'un agent

**Superviseur** : Trigger → AI Agent + Chat Model + Memory (historique) + Tools = {outil mémoire (Knowledge) + 1 Call-workflow par spécialiste}.

**Spécialiste** : 2 triggers (chat pour test + *Executed by Another Workflow* pour l'appel) → AI Agent + Chat Model + (Chat Memory avec fallback `sessionId`) + outil mémoire. Prompt utilisateur tolérant : `{{ query || chatInput }}`.

## 6. Gabarit de system message (spécialiste)

1. **Identité & domaine** (« You are <ROLE>, the <domain> specialist of <BRAND> »).
2. **Périmètre / boundaries** : rester dans son domaine, déférer aux autres spécialistes.
3. **Mémoire d'abord** : « Before answering, ALWAYS query the central memory with the "<ToolName>" tool. Never rely on assumptions. » (le `<ToolName>` doit matcher le **nom réel** du node-outil).
4. **Langue** : « Respond in the user's language. »
5. **Garde-fou humain** : marquer les actions critiques (coût, accès API, suppression, publication) comme nécessitant la validation du fondateur.
6. **Format de sortie** attendu (verdict / livrable structuré, précis et constructif).

## 7. Revue des agents — checklist

- [ ] Tous les agents **Published/Active** (les spécialistes doivent l'être pour être appelables).
- [ ] Nom de l'outil mémoire **cohérent** entre node et prompt.
- [ ] **Modèle** réellement configuré = celui documenté.
- [ ] **Chat Memory** des spécialistes : fallback `sessionId` (sinon plantage en sous-workflow).
- [ ] Prompts : langue + garde-fous humains présents.
- [ ] **Boucle de correction** réellement implémentée (re-fix jusqu'à validation humaine).
- [ ] **Déclencheurs/entrées-sorties** concrets définis (pas d'agents « à vide »).

## 8. Qualité du contenu de la mémoire

- Vérifier que le vector store contient bien ce que les agents doivent lire (ex. un agent « voix de marque » a besoin de la **bible de marque**, pas seulement de docs système).
- Séparer les sources d'autorité (bible/canon) du contenu de travail ; n'indexer que ce qui est validé/publié.

## 9. Sécurité & gouvernance

- MCP : **pas de « no auth »** hors MVP → OAuth2/clé.
- **Pas de secrets en clair** dans le chat ; secrets dans n8n / gestionnaire dédié.
- **Gating humain** sur les actions à effet (dépense, suppression mémoire, accès API, publication).
- Dépublier les workflows planifiés obsolètes.

## 10. Challenges récurrents & leçons

- Erreur opaque → toujours remonter au **node rouge** via Executions.
- Credential affichée OK mais binding mort → réassigner ; balayer **tous** les workflows.
- **TRUNCATE-puis-reload** sans garde-fou = perte de données sur échec.
- **SQL concaténé** = fragile sur contenu riche → paramètres.
- Auto-fermeture `{{ }}` dans les éditeurs d'expression → vérifier le rendu.
- Valider **end-to-end**, jamais seulement node par node.
- **Nommage** : séparer infra partagée et items de marque dès le départ.
