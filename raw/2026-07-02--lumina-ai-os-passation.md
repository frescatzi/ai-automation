---
type: raw
title: "LUMINA_AI_OS_Passation"
source_url: "drive:12knQ04aEmStr35STn2GG_MzPi9WPUvK0"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# LUMINA AI OS — Document de passation

**Projet :** LUMINA AI OS — plateforme multi-agents & multi-LLM (orchestration sur n8n)
**Objet :** Stratégie de répartition des tâches entre LLM, benchmark de routage réutilisable, et couche mémoire.
**Version :** 2.0 (benchmark fusionné + couche mémoire)
**Date de rédaction :** 2026-07-01
**Responsable projet :** Karter — info@iamkarter.com
**Statut :** Cadrage validé · MVP à implémenter dans n8n

---

## 1. Objectif du projet

Construire une couche d'orchestration qui **sélectionne automatiquement le meilleur LLM** selon la nature de chaque tâche, en arbitrant entre **qualité, coût, vitesse et confidentialité**.

Contrainte structurante : le système doit être **agnostique à la marque** (« brand-agnostic »). Un seul et même workflow doit tourner pour n'importe laquelle des marques du portefeuille, piloté par un fichier de configuration/JSON par marque — sans jamais dupliquer la logique.

Principe directeur : **on ne colle pas un agent à un seul LLM.** Les agents restent spécialisés par métier ; un *LLM Orchestrator (AI Router)* choisit dynamiquement le moteur à chaque étape du workflow.

---

## 2. Stack technique

| Brique | Choix | Rôle |
|---|---|---|
| Orchestration | **n8n** | Workflows, nœud Switch/Code de routage, crons |
| Déploiement / infra | **Coolify** sur **VPS** (en ligne) | Gère les conteneurs Docker et expose les endpoints |
| LLM premium (API) | **ChatGPT** (OpenAI), **Gemini** (Google), **Claude** (Anthropic) | Tâches à forte valeur |
| LLM local | **Hermes** (Nous) — auto-hébergé, servi via **Coolify** (⚠️ **pas** Ollama) | Volume, confidentialité, mémoire |
| Mémoire (à ajouter) | Base vectorielle — **Qdrant** ou **pgvector** (à trancher) | Rétention long terme, RAG |

> Note infra : l'endpoint qu'expose Coolify pour Hermes est l'URL appelée par le nœud n8n pour tout `task_type` routé vers Hermes. Endpoint OpenAI-compatible recommandé pour uniformiser les appels.

---

## 3. Profils & scoring des LLM

Scores indicatifs de 1 (faible) à 5 (excellent). **Le routage se fait sur le critère DOMINANT de chaque tâche, pas sur la moyenne.** À réviser à chaque nouvelle génération de modèle.

| Moteur | Rédaction | Raisonn. | Recherche web | Multimodal | Gén. image | Code | Coût | Vitesse | Confid. | Écosyst. agent |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| ChatGPT (OpenAI) | 4 | 5 | 4 | 4 | 5 | 5 | 2 | 3 | 2 | 5 |
| Gemini (Google) | 3 | 4 | 5 | 5 | 4 | 4 | 4 | 4 | 2 | 4 |
| Claude (Anthropic) | 5 | 5 | 3 | 4 | 1 | 5 | 2 | 3 | 3 | 4 |
| Hermes (Nous · self-hosted · Coolify) | 3 | 3 | 1 | 2 | 1 | 3 | 5 | 4 | 5 | 3 |

**Synthèse en une ligne par moteur :**

- **ChatGPT** — le généraliste-agent : raisonnement + écosystème d'outils le plus mature, génération d'images native. Faible sur coût et confidentialité.
- **Gemini** — le moteur recherche/contexte : ancrage web natif, multimodal (dont vidéo), contexte massif, tier Flash rapide et bon marché. Écriture plus plate.
- **Claude** — le rédacteur/analyste : meilleure plume et nuance de ton, raisonnement et code solides, sortie structurée fiable. Pas de génération d'image.
- **Hermes** — le cheval de trait privé : coût ≈ 0, données 100 % sur le VPS, orientable/fine-tunable. Qualité brute sous les modèles frontière.

---

## 4. Matrice de routage (activité business → LLM)

Réutilisable pour toutes les marques. La colonne `task_type` est la valeur pilotée par le nœud Switch de n8n.

| Domaine | Activité business | `task_type` | Primaire | Secours | Critère dominant |
|---|---|---|---|---|---|
| Recherche | Recherche d'info en ligne / veille | `research` | Gemini | ChatGPT | Recherche web |
| Recherche | Fact-checking / vérification finale | `verify` | Gemini | Claude | Qualité (contrôle) |
| Stratégie | Brainstorming / 1re ébauche | `draft` | ChatGPT | Claude | Qualité (divergence) |
| Stratégie | Réflexion stratégique / business plan | `reasoning` | Claude | ChatGPT | Qualité raisonnement |
| Analyse | Analyse de données / interprétation | `analysis` | Claude | ChatGPT | Qualité raisonnement |
| Analyse | Synthèse de docs longs / multi-fichiers | `summarize_long` | Gemini | Claude | Coût + contexte |
| Rédaction | Copy premium (article, blog, contenu) | `copy` | Claude | ChatGPT | Qualité rédaction |
| Rédaction | Email / comms partenaires | `copy` | Claude | ChatGPT | Qualité (ton) |
| Rédaction | Storytelling / narration de marque | `copy` | Claude | ChatGPT | Qualité rédaction |
| Communication | Traduction / localisation | `translate` | Gemini | Claude | Coût + couverture |
| Marketing | Idéation marketing / angles | `draft` | ChatGPT | Claude | Qualité (divergence) |
| Création | Génération d'images / visuels | `image_gen` | ChatGPT | Gemini | Capacité native |
| Développement | Code applicatif / architecture complexe | `code` | Claude | ChatGPT | Qualité code |
| Développement | Scripts n8n / SQL / Docker / workflows | `code` | ChatGPT | Claude | Écosystème + qualité |
| Automatisation | Classification / tagging en volume | `classify` | Hermes | Gemini Flash | Coût + confidentialité |
| Automatisation | Extraction / structuration de données | `extract` | Hermes | ChatGPT | Coût + confidentialité |
| Automatisation | Nettoyage / reformatage / métadonnées | `preprocess` | Hermes | — | Coût |
| Support | Support client L1 / réponses type | `support_l1` | Hermes | Claude | Confidentialité + coût |
| Confidentialité | Traitement de données sensibles / PII | `private` | Hermes | — | Confidentialité (non négociable) |
| Multimodal | OCR / analyse PDF / image / vidéo | `multimodal` | Gemini | ChatGPT | Qualité multimodale |
| Temps réel | Réponses à faible latence | `realtime` | Hermes / Gemini Flash | — | Vitesse |
| Orchestration | Routage / arbitrage entre agents | `orchestrate` | Claude | ChatGPT | Fiabilité function calling |
| Validation | Validation croisée d'un livrable critique | `verify` | Gemini | Claude | Qualité (contrôle) |

---

## 5. Règles de routage (arbre de décision, brand-agnostic)

À appliquer **avant** la matrice — c'est la logique qui généralise à toute nouvelle activité ou marque.

1. **Donnée sensible / PII en jeu ?** → **Hermes** (local). NON NÉGOCIABLE : la confidentialité écrase tout autre critère.
2. **Tâche à fort volume ET faible complexité ?** (classification, extraction, réponses type) → **Hermes** ; à défaut **Gemini Flash**. Le coût prime.
3. **Sinon, arbitrer selon la NATURE de la tâche :**
   - Rédiger / raisonner / nuancer → **Claude**
   - Chercher / synthétiser / multimodal → **Gemini**
   - Idéation / image / outillage-agent → **ChatGPT**
4. **Livrable complexe ou à fort enjeu ?** → **Chaîne multi-LLM** : Hermes (prépa données) → ChatGPT (création) → Claude (ciselage) → Gemini (validation).
5. **Validation croisée** → déclencher un 2e LLM **uniquement** si livrable externe/critique (contrat, comms partenaire, chiffres publics). Sinon on évite le coût x2.
6. **Vitesse critique (temps réel)** → Hermes ou Gemini Flash ; jamais un modèle premium sur du temps réel de masse.

---

## 6. Contrat de données — payload JSON n8n

Objet standardisé passé à chaque nœud d'agent. C'est le mécanisme de réutilisabilité : **1 workflow, N marques.**

```json
{
  "meta": {
    "brand_id": "BRAND_01",
    "execution_id": "exec_98765",
    "timestamp": "2026-07-01T00:00:00Z"
  },
  "routing": {
    "task_type": "copy",
    "preferred_engine": null,
    "fallback_engine": null,
    "cross_validate": false
  },
  "brand_profile": {
    "tone_voice": "Professionnel, minimaliste, transparent",
    "target_audience": "Partenaires B2B haut de gamme",
    "forbidden_words": ["révolutionnaire", "opportunité unique", "innovant"],
    "kb_ref": {
      "vector_collection": "brands/BRAND_01",
      "top_k": 6,
      "memory_scope": ["semantic", "episodic"],
      "min_score": 0.75,
      "entity_filter": "partenaire:X"
    }
  },
  "task_payload": {
    "context_raw": "Sortie préalable (ex : synthèse Gemini).",
    "instruction": "Rédiger l'email d'approche pour une intégration pilote."
  }
}
```

### Énumération `task_type`

| `task_type` | Moteur ciblé (défaut) | Note |
|---|---|---|
| `research` | Gemini | recherche / veille web |
| `verify` | Gemini | fact-check / validation croisée |
| `draft` | ChatGPT | brouillon / idéation |
| `reasoning` | Claude | stratégie / raisonnement |
| `analysis` | Claude | analyse de données |
| `summarize_long` | Gemini | synthèse gros volume |
| `copy` | Claude | rédaction finale premium |
| `translate` | Gemini | traduction / localisation |
| `image_gen` | ChatGPT | génération de visuels |
| `code` | Claude / ChatGPT | code (voir matrice §4) |
| `multimodal` | Gemini | OCR / PDF / image / vidéo |
| `classify` | Hermes | classification en volume |
| `extract` | Hermes | extraction de données |
| `preprocess` | Hermes | nettoyage / reformatage |
| `support_l1` | Hermes | support niveau 1 |
| `private` | Hermes | PII / données sensibles |
| `realtime` | Hermes / Gemini Flash | faible latence |
| `orchestrate` | Claude | routage entre agents |

---

## 7. Architecture d'orchestration (n8n)

Deux modes complémentaires :

**a) Routeur à choix unique** — pour les tâches simples, le nœud Switch lit `routing.task_type` et appelle un seul moteur.

```
[Entrée : données brutes de la marque]
        |
        v
[Nœud Switch / Code JavaScript]
        |
        ├─► research ──► [API Gemini] ──► synthèse
        ├─► draft ──► [API ChatGPT] ──► structure conceptuelle
        ├─► copy ──► [API Claude] ──► texte final ciselé
        └─► classify/private ──► [Hermes local via Coolify] ──► exécution privée
```

**b) Chaîne multi-LLM** — pour les livrables complexes (règle 4) :

```
Hermes (préparation) → ChatGPT (création) → Claude (amélioration) → Gemini (validation)
```

**Composant central — LLM Orchestrator (AI Router)**, responsable de : identifier la nature de la demande · sélectionner le moteur · enchaîner plusieurs modèles si besoin · arbitrer coût/vitesse/qualité · limiter les appels API quand Hermes suffit.

---

## 8. Couche Mémoire Hermes (rétention long terme)

Hermes est valorisé comme **hippocampe du système**. Étant local et gratuit à l'usage, il est le seul moteur qu'on peut faire tourner sur **100 % du trafic** en continu — impossible économiquement avec une API payante.

### Trois types de mémoire

| Type | Contient | Stockage |
|---|---|---|
| **Épisodique** | Chaque exécution, échange, livrable (log horodaté) | Base vectorielle + `brand_id` |
| **Sémantique** | Faits stables : voix de marque, mots interdits, personas, produits | Base vectorielle (collection consolidée) |
| **Procédurale** | Sorties acceptées / éditées / rejetées par moteur | Table de scoring → alimente la matrice §4 |

### Trois flux à câbler dans n8n

1. **WRITE** (fin de chaque workflow) : Hermes résume, extrait les entités, tague, génère l'embedding → stocké. Gratuit, donc systématique.
2. **READ / RAG** (avant l'action d'un agent premium) : interroge la base, renvoie le contexte pertinent → injecté dans `task_payload.context_raw` et `brand_profile.kb_ref`.
3. **CONSOLIDATION** (cron nocturne) : relit le brut, déduplique, fusionne, promeut les faits clés. L'épisodique devient sémantique.

### Boucle procédurale & fine-tuning

- **Feedback scoring (continu)** : Hermes observe le taux d'acceptation par moteur/`task_type` → met à jour les scores de la matrice. Le benchmark devient vivant.
- **Fine-tuning LoRA (tous les 1–3 mois)** : Hermes absorbe les meilleures données validées dans ses poids. Avantage exclusif de l'auto-hébergé → **actif propriétaire non réplicable**.

---

## 9. Historique des décisions

- **Format d'orchestration** : n8n confirmé (et non « N-10 » — coquille initiale).
- **Hermes** : identifié comme **Nous Hermes auto-hébergé**, servi via **Coolify** sur le VPS. ⚠️ **N'utilise PAS Ollama** (correction du 2026-07-01).
- **Rôle d'Hermes recentré** : classification / extraction / PII / mémoire — et **non** « exécution & sync système » comme le proposait la version Gemini.
- **Couloir code splitté** : architecture complexe → Claude ; scripts n8n/SQL/Docker → ChatGPT. À confirmer selon l'usage réel (voir §10).
- **Confidentialité** érigée en règle n°1 non négociable (angle mort de la version ChatGPT).
- **Benchmark fusionné** à partir de 3 sources indépendantes (Claude, ChatGPT, Gemini) → consensus unanime sur le socle : Gemini=recherche, ChatGPT=idéation, Claude=rédaction, Hermes=local.

---

## 10. Points ouverts à trancher

1. **Couloir code** : valider empiriquement si Claude ou ChatGPT est primaire sur le code applicatif, selon les stacks réellement utilisées.
2. **Base vectorielle** : Qdrant vs pgvector (à choisir + config Coolify).
3. **Seuils de la boucle procédurale** : à partir de quel taux d'édition on rétrograde un moteur dans la matrice.
4. **Politique de validation croisée** : définir précisément la liste des livrables « critiques » qui déclenchent un 2e LLM.
5. **Modèle d'embedding** : Hermes lui-même vs un modèle d'embedding dédié.

---

## 11. Roadmap

**MVP**
- Benchmark des LLM ✅ (ce document + fichier Excel)
- Règles de routage ✅
- Intégration dans n8n (nœud Switch + appels API) — à faire
- Validation manuelle des sorties — à faire

**Version avancée**
- Routage automatique piloté par le payload JSON
- Couche mémoire Hermes (write / read / consolidation)
- Auto-évaluation + validation croisée
- Historique des performances → matrice vivante
- Fine-tuning LoRA périodique d'Hermes

---

## 12. Prochaines étapes opérationnelles

1. **Isolation des contextes dans Coolify** : exposer un endpoint Hermes stable et sécurisé (OpenAI-compatible) accessible par l'orchestrateur n8n.
2. **Bibliothèque de méta-prompts système** : rédiger 1 prompt système par rôle — Gemini-Chercheur, ChatGPT-Créateur, Claude-Rédacteur, Hermes-Local — pour ancrer chaque spécialisation.
3. **Banque de connaissances par marque** : centraliser les fichiers de config de chaque marque (chartes, lexiques, personas, `forbidden_words`) en JSON statiques ou base vectorielle légère consultables à la volée.
4. **Déployer la base vectorielle** (Qdrant/pgvector) via Coolify + brancher les nœuds n8n write/read.

---

## 13. Livrables & références

- `LUMINA_AI_OS_Benchmark_Routage_LLM_v2.xlsx` — benchmark scoré, matrice, règles, payload, couche mémoire (5 onglets).
- `LUMINA_AI_OS_Passation.md` — le présent document.

**Contact projet :** Karter — info@iamkarter.com
