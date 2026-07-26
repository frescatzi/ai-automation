---
type: raw
title: "CARTOGRAPHIE_Automatisation-n8n-Notion_Evenementiel-et-Hors-evenementiel_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/cartographie_automatisation-n8n-notion_evenementiel-et-hors-evenementiel_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# CARTOGRAPHIE : Scénarios d'automatisation n8n × Notion

**Marque :** AFTER SUN PEOPLE (AFTRSN) · **Date :** 17.07.2026 · **Auteur/validateur :** Karter
**But :** figer la liste des scénarios automatisables, partie **événementielle** et partie **hors-événementielle**, en distinguant pour chaque étape : ce qui **tourne seul**, ce qui se fait **en parallèle**, ce qui exige **ma validation** et ce qui **n'en a pas besoin**. Une fois ce document validé, il sert de plan d'entrée pour la session de build n8n.

> ⚠️ **Note d'honnêteté :** la discussion d'origine où le process avait été déroulé n'a pas été retrouvée en archive (ni projet, ni mémoire, ni dossier local). Ce document est une **reconstruction** à partir de ta description + des structures déjà en place (Backstage Notion, agent Channel Content, modèle de validation « Karter valide les actions critiques »). À corriger là où la réalité diffère.

---

## 1. Principes directeurs (hérités de l'existant)

Trois règles déjà actées dans le projet encadrent toute la cartographie :

1. **Le monde « Human » et le monde « Agent » cohabitent sur une même base.** Dans Backstage v3, la propriété `Executor` (🧑 Human / 🤖 Agent) partitionne les tâches sans les dupliquer. Une automatisation = une tâche `Executor = 🤖 Agent`.
2. **Gate de validation sur les actions critiques.** Par défaut on donne de l'autonomie aux agents, MAIS toute action **coûteuse, externe ou irréversible** passe par ma validation : dépense d'argent, envoi/publication vers l'extérieur, suppression, accès API sensible. (Décision CEO/MVP.)
3. **Notion = source de vérité et déclencheur.** La définition d'une **Edition** (date, format, ville, venue) est le point d'entrée. n8n écoute Notion, produit, et repose ses résultats en `To Validate` pour les actions critiques.

### Légende des niveaux d'automatisation

| Niveau | Sens | Validation ? |
|---|---|---|
| 🟢 **Auto** | Tourne seul de bout en bout (préparation, données internes, brouillons en staging) | Non |
| 🟡 **Auto + gate** | Produit automatiquement, mais s'arrête avant l'action finale en attendant mon OK | Oui, avant publish/send/dépense |
| 🔴 **Humain** | Décision créative ou sensible : je fais (l'agent peut assister/suggérer) | N/A (c'est moi) |

> La brique « gate » existe déjà : le pattern **M8 Validation Gate** (boutons inline, approbation owner, expiration) construit pour Lyra est **réutilisable tel quel** pour approuver un post, une newsletter ou une campagne.

---

## 2. Partie événementielle : la chaîne de A à Z

### 2.1 Vue d'ensemble (le flux)

```
[P1] Date d'événement définie (Notion Edition)   ← DÉCLENCHEUR
        │
        ▼
[P2] Brief créatif consolidé (nom, date, venue, line-up, tagline, visuel-clé)
        │
        ├──────────────┬──────────────┬───────────────┬─────────────┐   ← EN PARALLÈLE
        ▼              ▼              ▼               ▼             ▼
[P3] Visuels Canva  [P4] Site Wix  [P5] Newsletter  [P6] Posts IG  [P7] Facebook Ads
 (flyer/story/web)   (page event)   (annonce)        (feed+story)   (campagne)
        │              │              │               │             │
        └──────────────┴──────┬───────┴───────────────┴─────────────┘
                              ▼
                    [GATE] Validation Karter (publish / send / dépense)
                              ▼
                    [P8] Diffusion effective + suivi
                              ▼
                    [P9] Post-événement (aftermovie, débrief, KPI)
```

**Lecture clé :** une fois le **brief (P2)** figé, les cinq chantiers de production (P3→P7) sont **indépendants et parallélisables** (chacun son agent/sous-workflow). Ils **convergent tous** sur une **porte de validation unique** avant toute sortie vers l'extérieur.

### 2.2 Détail étape par étape

#### P1 : Définition de la date d'événement (déclencheur)

| Élément | Détail |
|---|---|
| Ce qui se passe | Création d'une **Edition** dans Notion : date, format (Daytime / Day-to-Night / Nightclub), ville, venue, budget cap. |
| Décision | 🔴 **Humain** : la date, le concept, le line-up sont ta décision stratégique. |
| Automatisable | 🟢 Sur création/passage d'une Edition à un statut « Go », n8n **génère automatiquement la checklist de tâches** de l'édition + un objet **brief** vide à remplir. |
| Parallèle / séquentiel | Point de départ, séquentiel par nature. |
| Validation | Non (c'est toi qui crées l'Edition). |

#### P2 : Brief créatif consolidé

| Élément | Détail |
|---|---|
| Ce qui se passe | On rassemble en un seul endroit tout ce dont les 5 chantiers ont besoin : nom d'édition, date/heure, venue + adresse, line-up, tagline/accroche, **visuel-clé (artwork)**, lien billetterie. |
| Visuel-clé (artwork) | 🔴 **Humain** (ou IA-assisté mais validé) : c'est l'ADN visuel, il ne se délègue pas à l'aveugle. |
| Textes/copy de base | 🟡 L'agent **Channel Content** rédige les accroches et déclinaisons, tu valides la version de référence. |
| Automatisable | 🟢 Compilation du brief depuis les champs Notion de l'Edition. |
| Parallèle / séquentiel | **Séquentiel** : c'est le verrou, rien ne part en production tant que le brief (surtout artwork + tagline) n'est pas figé. |
| Validation | 🟡 Validation du brief = le **premier point de contrôle** (léger). |

#### P3 : Production visuelle Canva (flyer · story · web)

| Élément | Détail |
|---|---|
| Ce qui se passe | Déclinaison de l'identité de l'événement en 3 formats : **flyer** (post carré/print), **story** (9:16), **affichage web** (bannière). |
| Automatisable | 🟡 Avec des **Brand Templates Canva + Autofill API** : une fois artwork + textes fixés, n8n **remplit les 3 gabarits automatiquement** et génère les exports. |
| Parallèle / séquentiel | Les **3 formats en parallèle** entre eux, le bloc P3 en parallèle de P4 à P7. |
| Validation | 🟡 **Gate visuel** : tu valides le rendu avant qu'il serve ailleurs (un mauvais gabarit se propage partout sinon). |
| Dépendance | A besoin de l'artwork + tagline (P2). |

#### P4 : Site web événement (Wix)

| Élément | Détail |
|---|---|
| Ce qui se passe | Création/mise à jour de la **page événement** sur Wix : titre, date, description, visuel web, lien billetterie. |
| Automatisable | 🟡 Via l'API Wix, n8n **construit la page en brouillon** (draft) à partir du brief + du visuel web. |
| Parallèle / séquentiel | En parallèle des autres chantiers, **dépend du visuel web (P3)** pour l'illustration → léger chevauchement. |
| Validation | 🟡 **Gate publication** : la mise en ligne du site est une action externe, tu valides puis publies. |

#### P5 : Newsletter (annonce d'événement)

| Élément | Détail |
|---|---|
| Ce qui se passe | Rédaction d'une newsletter d'annonce : visuel + copy + lien billetterie + call-to-action. |
| Automatisable | 🟢 Rédaction/montage du brouillon (agent Channel Content, skill `email-sequence`). |
| Parallèle / séquentiel | En parallèle. |
| Validation | 🔴/🟡 **L'ENVOI = action critique** → gate obligatoire avant expédition à la base d'abonnés. |
| ⚠️ Prérequis | **Outil d'emailing à confirmer** (Mailchimp / Brevo / autre / natif Wix), voir section 4. |

#### P6 : Posts Instagram (feed + story)

| Élément | Détail |
|---|---|
| Ce qui se passe | Post feed (flyer + légende + hashtags) et story (visuel story + sticker lien). |
| Automatisable | 🟡 Génération du visuel (P3) + rédaction légende/hashtags automatique + **programmation**. |
| Parallèle / séquentiel | En parallèle. |
| Validation | 🔴 **La PUBLICATION = action critique** → gate obligatoire. Ton compte Insta porte une **charge personnelle/émotionnelle** (cf. historique des stories) : la validation humaine est ici particulièrement importante, pas seulement pour l'événementiel. |
| ⚠️ Prérequis | Compte **Instagram Business/Creator** lié à une Page Facebook + permissions Graph API (voir section 4). |

#### P7 : Facebook Ads (campagne payante)

| Élément | Détail |
|---|---|
| Ce qui se passe | Campagne Meta : audience, budget, créa (flyer/web), objectif (vente billets / notoriété), landing = site ou billetterie. |
| Automatisable | 🟡 n8n **prépare la campagne** (structure, audiences, créas) via l'API Marketing Meta. |
| Parallèle / séquentiel | En parallèle, mais **dépend** des créas (P3) ET d'une landing prête (P4/billetterie). |
| Validation | 🔴 **DÉPENSE D'ARGENT = gate le plus strict.** Rien ne se lance sans mon OK explicite sur le budget. C'est le scénario à plus haut risque → jamais en auto complet. |
| ⚠️ Prérequis | **Facebook Marketing API** + Business Manager + moyen de paiement + revue d'app Meta (voir section 4). |

#### GATE : Porte de validation unique

Tous les chantiers déposent leurs livrables en `To Validate` (côté Notion) et/ou déclenchent une **demande d'approbation** (pattern M8 : bouton Valider / Réviser sur Telegram ou email). Un seul endroit pour dire oui/non/à revoir. Rien de coûteux ou d'externe ne franchit cette porte sans moi.

#### P8 : Diffusion effective + suivi

| Élément | Détail |
|---|---|
| Après validation | 🟢 n8n exécute réellement : publie le post, envoie la newsletter, met le site en ligne, active la campagne. |
| Suivi | 🟢 Collecte automatique des premiers indicateurs (portée, clics, ventes) → tableau KPI Notion. |
| Validation | Déjà donnée au GATE, l'exécution est mécanique. |

#### P9 : Post-événement

| Élément | Détail |
|---|---|
| Ce qui se passe | Aftermovie, débrief, recap KPI, remerciements. |
| Automatisable | 🟢 Recap KPI + brouillon de débrief (agent **Acquisition Performance**, qui fait déjà les recaps). 🟡 Post de remerciement (gate publication). 🔴 Montage aftermovie = humain/créatif. |
| Validation | 🟡 sur les sorties publiques. |

### 2.3 Synthèse événementiel

| Ce qui peut tourner **100 % auto** (🟢) | Ce qui est **auto mais attend ma validation** (🟡/🔴) | Ce qui reste **humain** (🔴) |
|---|---|---|
| Génération checklist + brief sur nouvelle Edition · compilation des données · brouillons (copy, légendes, newsletter) · génération des 3 visuels Canva en staging · construction page Wix en draft · collecte KPI · recap de débrief | Publication site (P4) · envoi newsletter (P5) · publication posts IG (P6) · **lancement Facebook Ads / budget (P7, le plus strict)** · sorties publiques post-event | Concept + date + line-up (P1) · artwork/visuel-clé (P2) · montage aftermovie (P9) |

**Parallélisme :** après P2, les 5 chantiers (P3→P7) tournent **en parallèle**, ils **convergent** sur la porte de validation avant diffusion.

---

## 3. Partie hors-événementielle : le récurrent

Ici, pas de date d'événement : c'est le flux permanent du label/de la marque. On le range en 5 streams (cohérents avec les catégories Office déjà présentes dans Backstage : Admin, Legal, Finances, Prospection, Brand & Community).

### 3.1 Stream A : Contenu label / musique (Instagram)

| Élément | Détail |
|---|---|
| Ce qui se passe | Stories & posts récurrents : jalons (« +60k auditeurs »), promo de mix/release, teasers. |
| Automatisable | 🟢 Idées/brouillons + **calendrier éditorial** + rappels de publication. 🟡 Génération de visuels récurrents (templates Canva). |
| Validation | 🔴 **Charge personnelle et sensibilité de ton compte** → publication toujours validée par toi. Automatiser le rythme et la préparation, pas le jugement. |
| Parallèle | Indépendant du reste. |

### 3.2 Stream B : Community management

| Élément | Détail |
|---|---|
| Ce qui se passe | Réponses DM/commentaires, engagement. |
| Automatisable | 🟡 **Tri + suggestions de réponses** (l'agent propose, tu envoies). 🟢 Détection de messages prioritaires. |
| Validation | 🔴 La relation/le ton = humain. Pas d'envoi automatique. |

### 3.3 Stream C : Newsletter récurrente (hors event)

| Élément | Détail |
|---|---|
| Ce qui se passe | News label périodique (releases, coulisses, agenda). |
| Automatisable | 🟢 Compilation + brouillon. |
| Validation | 🔴 **Envoi = gate obligatoire.** |

### 3.4 Stream D : Office / Admin / Business

| Élément | Détail |
|---|---|
| Suivi budget/finances | 🟢 Agrégation, mises à jour de vues, **alertes de dépassement** (déjà envisagé dans Backstage). |
| Prospection (venues, sponsors, artistes) | 🟡 Recherche + **brouillon d'emails** de prospection, 🔴 **envoi validé**. |
| Legal / contrats | 🟢 Rappels d'échéance, 🔴 signature/engagement = humain. |
| Paiements (cachets, abonnements) | 🔴 **Dépense = gate strict**, toujours. |

### 3.5 Stream E : Veille / recherche

| Élément | Détail |
|---|---|
| Ce qui se passe | Veille tendances, repérage venues/artistes, benchmark. |
| Automatisable | 🟢 Recherche web (agent Web + Hermès) → notes/synthèses en Notion. |
| Validation | Non (interne, pas d'action externe). |

### 3.6 Synthèse hors-événementiel

| 🟢 Auto | 🟡 Auto + gate | 🔴 Humain |
|---|---|---|
| Calendrier éditorial · brouillons · suivi budget + alertes · rappels échéances · veille/recherche · tri community | Génération visuels récurrents · brouillons prospection · suggestions de réponses | **Toute publication IG** · envoi newsletter · réponses community · signatures/contrats · **tout paiement** |

---

## 4. Prérequis & connecteurs à trancher AVANT le build n8n

Ce sont les points qui déterminent la faisabilité réelle de chaque scénario :

| Chantier | Connecteur / condition | Statut |
|---|---|---|
| Déclencheur + source de vérité | **Notion API** (intégration `AFTRN-n8n` déjà existante) | ✅ En place |
| Visuels | **Canva Connect API** (Brand Templates + Autofill). Nécessite un plan Canva compatible API | ❓ À vérifier (plan + accès dev) |
| Site | **Wix API/Headless** | ✅ Connecteur Wix disponible, à câbler |
| Newsletter | **Outil d'emailing** (lequel ?) | ❓ **À décider** (Mailchimp / Brevo / natif Wix / Gmail) |
| Instagram | **Instagram Graph API** (compte Business/Creator + Page FB liée + permissions) | ❓ À configurer (revue Meta) |
| Facebook Ads | **Facebook Marketing API** + Business Manager + moyen de paiement | ❓ À configurer (revue Meta, le plus lourd) |
| Porte de validation | Pattern **M8 Validation Gate** (Telegram/email, boutons inline) | ✅ Réutilisable tel quel |
| Cerveau rédactionnel | Agent **Channel Content** (skills `draft-content`, `email-sequence`, `canva-creator`) | ✅ Existe |

### Questions ouvertes à me poser (à figer avec toi)

1. **Newsletter :** quel outil d'emailing ? (détermine tout le chantier P5/Stream C.)
2. **Instagram :** publication **auto via API** (compte Business obligatoire) **ou** simple **programmation/rappel** et tu postes à la main ? (impacte P6/Stream A.)
3. **Billetterie :** où ? (Wix Events ? plateforme externe ?), c'est la landing des Ads.
4. **Canva :** ton plan permet-il l'API + les Brand Templates ? Sinon P3 reste manuel (Canva reçoit juste un brief).
5. **Facebook Ads :** on automatise dès le MVP, ou on garde 100 % manuel au début (vu le risque budget) et on branche plus tard ?

---

## 5. Ce qui est prêt pour la session n8n (handoff)

- **Périmètre figé** : 9 phases événementielles (P1→P9) + 5 streams hors-événementiel, chacun étiqueté auto / parallèle / validation.
- **Règle d'or** : tout ce qui **dépense, envoie, publie ou supprime** passe par la **porte de validation** (M8, déjà construite).
- **Build 100 % par API** : depuis le 17.07, le serveur MCP local `n8n-mcp` expose l'API n8n complète (création/modification/validation). Fini le build navigateur node par node (navigateur réservé aux credentials/OAuth).
- **Point d'entrée technique** : trigger Notion sur Edition → fan-out parallèle vers les chantiers → convergence sur le gate → exécution.
- **À trancher d'abord** : les 5 questions de la section 4 (surtout newsletter, Instagram, Ads), elles conditionnent quels workflows on peut réellement bâtir.
- **Prochaine session** : transformer cette cartographie en **spec de build n8n** (`SPEC-BUILD_Automatisation-Evenementielle-P3_2026-07-17.md`), en commençant par le(s) scénario(s) 🟢 les moins risqués pour valider la tuyauterie avant de brancher le payant.

---

*Document de travail à valider/corriger par Karter. Une fois figé, sert d'entrée à la spec de build n8n.*
