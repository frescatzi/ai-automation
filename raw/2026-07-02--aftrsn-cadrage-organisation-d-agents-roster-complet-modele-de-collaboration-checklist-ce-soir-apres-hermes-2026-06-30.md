---
type: raw
title: "AFTRSN — Cadrage organisation d'agents (roster complet + modele de collaboration + checklist ce soir apres Hermes) — 2026-06-30"
source_url: "drive:1S0diWHrIiLjOFxhABq8gOnJ1o5efhURz"
captured: 2026-07-02
vault: brands
brand: AFTRSN
immutable: true
---

# Cadrage — Organisation d'agents AFTER SUN PEOPLE (roster + collaboration + checklist « ce soir, après Hermes »)

**Date :** 2026-06-30 (fin d'après-midi)
**Statut :** cadrage figé. La **construction/redéfinition** se fait **ce soir**, **après la config de Hermes sur Coolify**.
**Autorité finale :** Karter (décision, demande de révision, recadrage).

> Convention : explications en français ; noms d'agents/nodes/outils en anglais (comme dans n8n).

---

## 1. Décisions validées

- **Le Superviseur** = assistant direct de Karter qui lui fait le **compte rendu** ; il est aussi **expert en curation & gestion événementielle**. Point d'entrée unique, point de sortie unique vers Karter.
- **Karter = facteur de décision final** : validation, demande de révision, recadrage lui reviennent.
- **Principe clé : les agents communiquent entre eux et peuvent travailler en parallèle.**
- **Mémoire :** Superviseur = `lumina_memory` (process partagé) **+ lecture `aftrsn_memory`** (contexte marque) ; spécialistes & experts = `aftrsn_memory`.
- **3 agents-experts (challengers) validés** : Music & Sound Curator · Culture & Community Steward · Experience & Event Designer.

## 2. Roster complet (org par niveaux)

**Niveau 0 — Direction**
- **Karter** — décision finale, révision, recadrage (humain dans la boucle).
- **AFTRSN-Supervisor** — orchestrateur + assistant de Karter + reporting + expert curation/événementiel.

**Niveau 1 — Experts de domaine (challengers, validés)**
- **Music & Sound Curator** — Afro House / Afro Tech / Melodic Afro House : crédibilité sonore, artistes, line-up.
- **Culture & Community Steward** — culture électronique africaine, diaspora, authenticité anti-cliché, rituel/communauté.
- **Experience & Event Designer** — le rituel jour/nuit en live : lieu, atmosphère, parcours du public.

**Niveau 2 — Fonctions business**
- **Marketing specialist (events)** — marketing orienté événementiel.
- **Comptable & Financier** — comptabilité, finances.
- **Controller — Qualité & Compliance** — contrôle, qualité, conformité.
- **Secrétaire** — rédaction des mails, vérification du calendrier/agenda.

**Niveau 3 — Opérations & utilitaires**
- **Web & Recherche** — piloter le web, faire des recherches.
- **Fichiers** — ouvrir, traiter, classer, sauvegarder les fichiers.
- **Catch-all « autres tâches business »** — agent(s) pour les tâches non couvertes / oubliées.

> Rôle des experts : ils ne *produisent* pas de livrable, ils **challengent** depuis leur domaine (idée « agents qui se contredisent »). La *connaissance* de domaine vit dans `aftrsn_memory` (lue par tous), via des `collection` dédiées (`music`, `culture`, `events`…) à alimenter.

## 3. Modèle de collaboration — à trancher CE SOIR

Deux options (les agents communiquent dans les deux, la différence est le *degré*) :

- **Orchestré + appels ciblés (recommandé)** — le Superviseur reste chef d'orchestre et point de contact unique ; chaque agent peut **appeler un autre agent comme outil** quand c'est utile ; le Superviseur peut **lancer plusieurs agents en parallèle** (fan-out). → collaboration + parallèle, **coût maîtrisé, débogable**.
- **Maillage large (peer-to-peer)** — chaque agent parle directement à tous les autres. → collaboration spontanée mais **coût tokens élevé**, risque de boucles, **difficile à tracer**, plus de point de compte rendu unique.

**Garde-fous (dans les deux cas) :** plafonner la profondeur des appels agent→agent ; **Karter valide les actions critiques** (coût, envoi, publication, suppression, accès API).

**→ DÉCIDÉ (2026-06-30, Karter) : « orchestré + appels ciblés ».** Superviseur = chef d'orchestre + point de compte rendu unique ; agents qui s'appellent via *Call n8n Workflow Tool* quand utile ; fan-out parallèle ; Hermes = bras opérationnel (Web/Fichiers/commandes). Maillage large rejeté (coût/débogage).

## 4. Dépendances & inconnues

- **Hermes (Coolify) — à configurer ce soir.** Plusieurs agents en dépendent : **Secrétaire (mails), Fichiers, Web**. À clarifier : **ce que Hermes couvre exactement** (mails ? fichiers ? automatisations ?) pour brancher ces agents au bon endroit.
- **Séquencement (1ʳᵉ vague) — à décider ce soir.** 11+ agents = trop pour une session. Proposition de 1ʳᵉ vague : **Superviseur + 3 experts + Secrétaire + Web/Fichiers** (socle assistant + curation), puis 2ᵉ vague : Marketing, Comptable/Finance, Qualité/Compliance.
- **Chevauchement à arbitrer :** le Superviseur est déjà expert curation/événementiel → préciser la frontière avec l'**Experience & Event Designer** (Superviseur = stratégique/curation ; Expert = design expérientiel détaillé).

## 5. Checklist « ce soir, après Hermes »

1. [ ] Configurer **Hermes sur Coolify** + noter sa portée (mails/fichiers/auto).
2. [ ] Trancher **orchestré vs maillage** (§3).
3. [ ] Choisir la **1ʳᵉ vague** d'agents (§4).
4. [ ] **Redéfinir tous les rôles** (périmètre, boundaries, ce qui est délégué à qui).
5. [ ] **Écrire les system prompts** (identité, « always query Knowledge », langue, format, garde-fous, validation Karter).
6. [ ] **Câbler la mémoire** : spécialistes/experts → `brand=aftrsn` ; Superviseur → `lumina` + lecture `aftrsn`.
7. [ ] Brancher les **outils** (Call-workflow entre agents, accès Hermes/Web/Fichiers selon l'agent).
8. [ ] Tester end-to-end + **compte rendu au format Karter**.

## 6. Rappels techniques (acquis aujourd'hui)

- Récupération mémoire = `Webhook (When Last Node Finishes / All Entries)` **sans** node `Respond to Webhook` ; MCP `search_brand_memory(brand=…)`.
- Banques : `lumina_memory` (partagé, ~66) · `aftrsn_memory` (marque, bible ingérée, `collection='canon'`).
- Outil mémoire des agents = **`Knowledge`** (nom node = nom dans le prompt).
- n8n : champs Expression auto-ferment `{{ }}` (nettoyer le ` }}` en trop) ; pour appeler un agent, « Call n8n Workflow Tool » + trigger « When Executed by Another Workflow » côté agent appelé.
