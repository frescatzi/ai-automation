---
type: raw
title: "CONCEPT_Backstage-v3_Deux-Mondes_2026-07-16"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/concept_backstage-v3_deux-mondes_2026-07-16.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# CONCEPT — Backstage v3 : « Deux mondes, un portail »

Date : 16.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **CONCEPT VALIDÉ par Karter — AMENDÉ le 16.07.2026 au soir (cockpit sur l'accueil, voir §0) — exécution en cours (N1+N2 faits)**
Contenu Notion en anglais ; ce document de référence en français.

---

## 0. AMENDEMENT du 16.07.2026 (décision Karter, après N2)

En voyant The Backstage vidé de son contenu opérationnel (état intermédiaire prévu avant N4), Karter a tranché : **« Le cockpit peut rester sur la page d'accueil Backstage. »**

Le Backstage n'est donc plus un « portail pur » mais un **« portail + cockpit »** : la page d'accueil EST le poste de pilotage quotidien, et les deux portes donnent sur les mondes. La file **✅ Awaiting validation vit aussi sur l'accueil** (choix explicite de Karter — tout le poste de pilotage au même endroit). Human Space se recentre sur Operations + Knowledge + Team. Le reste du concept (Executor, zéro duplication, pont de validation, Agent Space) est inchangé. Exécuté et validé le 16.07 au soir.

## 1. La vision (amendée)

Quand on entre dans The Backstage, on a sous les yeux **le travail du moment** (cockpit) et **deux endroits où aller** : on est un agent → Agent Space ; on est un humain → Human Space. Chaque monde reste complet : il contient son **travail** et sa **connaissance**. L'humain voit où les agents interviennent sans quitter ses vues (pastille Executor) et peut aller inspecter leur monde par la porte Agents.

Principe fondateur conservé de la v2 : **une seule mécanique, zéro duplication**. Les bases ne bougent que d'emplacement ; aucune n'est copiée.

## 2. Structure cible (amendée 16.07)

```
🎭 The Backstage  =  PORTAIL + COCKPIT (la page d'accueil pilote le quotidien)
│   callout identité
│   🗂️ Zones : 2 portes (🧑 Human Space · 🤖 Agent Space)
│   🔭 Cockpit : base Tasks + vues ⏭️ Up Next, 🎪 Upcoming Editions,
│                📣 Publishing Pipeline + file ✅ Awaiting validation
│   🗄️ Archive, discret en bas (le legacy n'appartient à aucun monde)
│
├── 🧑 HUMAN SPACE — le monde humain
│     ├─ ⚙️ Operations     les 10 bases, déménagées TELLES QUELLES
│     │                    (Editions, Débriefs, Sponsors, Venues, DJs, Cities,
│     │                    Budget & Finance, Content, KPI, Media)
│     ├─ 📚 Knowledge      wiki humain (SOP, Brand, How-to, Decisions, Reference)
│     │                    structure créée N2 ; rapatriement GitHub = phase 2
│     └─ 👥 Team           (existant)
│
├── 🤖 AGENT SPACE — le monde agent
│     ├─ 📋 Activity        vues Tasks filtrées Executor = Agent (board par statut)
│     ├─ ⏳ To validate     Executor = Agent & Status = To Validate
│     └─ 📚 Knowledge base  l'existant : roster 8 agents, règles, Brand Source
│                           of Truth, playbooks
│
└── 🗄️ Archive (sous le portail)
```

## 3. Le contrat de données

Deux propriétés portent toute la séparation :

| Axe | Propriété | Valeurs | Statut |
|---|---|---|---|
| Qui exécute ? | **Executor** (select sur Tasks) | 🧑 Human *(défaut)* · 🤖 Agent | ✅ CRÉÉ (N1) |
| Quel monde thématique ? | Relation **Event** (+ formule Section) | remplie = Events · vide = Office | Existant |

Décisions associées :
- **Pas de copie de tâches** vers Agent Space (validé) : une tâche où un agent intervient s'affiche automatiquement dans Agent Space via la vue filtrée Executor = Agent. Une seule vérité.
- Le template « New Task » met Executor = Human par défaut (✅ fait N1, par API).
- Si un jour il faut savoir *quel* agent : propriété « Agent » optionnelle (hors MVP).
- Côté humain, la colonne/pastille Executor 🤖 rend visible l'intervention des agents dans les vues humaines.

## 4. Le pont entre les mondes

La **validation** est le point de contact : un agent produit → tâche To Validate → visible à la fois dans Agent Space (⏳) et dans la file ✅ Awaiting validation du cockpit (page d'accueil) → l'humain valide → Done. La délégation inverse (assigner Executor = Agent depuis Notion et laisser n8n/Maestro exécuter) = **phase 2, hors périmètre**.

## 5. Knowledge — la symétrie des connaissances

- **Agents** : la knowledge base existe (page Agents actuelle : roster, Brand Source of Truth, playbook, règles Hermès).
- **Humains** : ✅ créée en N2 — base « Knowledge » dans Human Space (SOP, Brand, How-to, Decisions, Reference), première entrée 📘 Budget How-To (référence le SOP, pas de copie). Le **rapatriement automatique GitHub → Notion** sera un workflow n8n en phase 2.

## 6. Garde-fous (ce qu'on ne fait PAS)

- Pas de deuxième base de tâches, pas de copies.
- On ne touche pas au modèle relationnel des 10 bases (elles déménagent, elles ne changent pas).
- Pas de pont n8n↔Notion au MVP (ni délégation, ni sync wiki).
- Le workflow de statuts reste unique : To Do → In Progress → To Validate → Done (+ Blocked).

## 7. Plan d'exécution (état au 16.07 soir)

1. ✅ **N1** — propriété Executor (Human/Agent, défaut Human via template) + 4 tâches exemple taguées + pastille dans les vues.
2. ✅ **N2** — Human Space : Operations déménagée, 📚 Knowledge créé, page réorganisée. *(Le Cockpit, d'abord créé en sous-page, a été remonté sur l'accueil suite à l'amendement §0 — la sous-page vide a été corbeillée.)*
3. **N3** — Agent Space : remodeler la page Agents → sections Activity (vue Executor=Agent), To validate, Knowledge base.
4. **N4 (allégé par l'amendement)** — vérifier/polir la page d'accueil : callout + Zones (2 portes) + Cockpit + Archive — état déjà largement atteint.
5. **N5** — nettoyage sidebar (renommer les wrappers « View of… »).
6. **N6** — vérification bout-en-bout + passation.

⚠️ Points de vigilance : `notion-move-pages` déménage pages/databases/linked views (vérifié N2) ; re-vérifier après ~1 min ; une vue « vide » post-move = souvent cache navigateur → reload ; `replace_content` doit relister tous les blocs enfants ; ne jamais `in_trash` une data source partagée ; deux « The Backstage » existent (AFTRSN HQ + LUMINA).

## 8. Hors périmètre (phase 2, plus tard)

- Délégation Notion → n8n/Maestro (file de travail agents pilotée depuis Notion).
- Sync GitHub → Notion pour le wiki humain.
- Propriété « Agent » (quel agent précisément).
- Executor/équivalent sur d'autres bases que Tasks (ex. Content & Social).
