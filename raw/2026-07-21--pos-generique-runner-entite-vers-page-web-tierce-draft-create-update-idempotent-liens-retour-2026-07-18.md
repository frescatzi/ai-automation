---
type: raw
title: "POS-GENERIQUE_runner-entite-vers-page-web-tierce-draft-create-update-idempotent-liens-retour_2026-07-18"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-entite-vers-page-web-tierce-draft-create-update-idempotent-liens-retour_2026-07-18.md"
captured: 2026-07-21
vault: ai-automation
brand: null
immutable: true
---

# POS-GÉNÉRIQUE · Runner « entité Notion → page web tierce en draft » (create/update idempotent, link-back, jamais publié)

**Date :** 18.07.2026 · **Patron dérivé de :** M4 Wix Event Runner (AFTRSN P3) + refactor fiabilité M2. Réutilisable pour tout service tiers (Wix, Webflow, Shopify, un CMS…) piloté par API depuis n8n × Notion.

---

## 1. Quand utiliser ce patron
Un contenu produit dans Notion (copy validée à venir) doit se matérialiser en **page/objet sur une plateforme externe**, en **brouillon** (rien de public sans OK humain), avec :
- **création** au premier contenu, **mise à jour** aux suivants (une seule page par entité, jamais de doublon),
- un **lien retour** (l'ID + l'URL externes réécrits sur l'entité Notion),
- des **variantes** selon un attribut (ex. gratuit → inscription ; payant → lien externe),
- des **garde-fous** : contenu incomplet ou entité gelée ⇒ tâche `Blocked` + raison, jamais d'écriture partielle publiée.

## 2. Schéma (additif, zéro casse)
Sur l'entité source, ajouter :
- **Attribut de variante** (select) qui pilote la logique (ex. Free/Paid).
- **Champ(s) d'entrée conditionnels** (ex. URL externe requise si « Paid »).
- **`<Service> ID`** (text) + **`<Service> draft URL`** (url) : **écrits par l'automate** = link-back + **clé d'idempotence** (présent ⇒ update, absent ⇒ create).
- Un **statut** qui **gèle** l'entité au-delà d'un certain point (ex. Live/Cancelled/Done ⇒ aucune modif). Le garde-fou des modifications, c'est le **statut**, pas une date.

## 3. Squelette du workflow
```
[Déclencheur] → Read source → Should build? (extrait champs + id entité ; skip si vide)
   → Read entity → Prepare (décide route: create | update | blocked ; construit la query de tâche)
   → Find task (HTTP query Notion) → Attach task
   → IF route==update → Build update → PATCH service → Mark task To Validate
   → ELSE IF route==create → Build create (draft:true) → POST service → Parse (id+url)
         → PATCH entity (écrit <Service> ID + draft URL) → Mark task To Validate
   → ELSE → Mark task Blocked (Status + Notes=raison)
```
- **Idempotence** : la présence de `<Service> ID` route en update → jamais de recréation. Les updates sont **partiels** (n'envoyer que les champs présents).
- **Draft obligatoire** (`draft:true` / statut brouillon) : la publication reste un geste humain (gate).
- **HTTP Request générique** pour l'API tierce (auth Custom/OAuth par credential) ; **HTTP Request → API Notion** quand le node Notion ne suffit pas (query filtrée, PATCH page).

## 4. Fiabilité du déclenchement (leçon centrale)
**Ne pas dépendre du trigger Notion « Updated ».** Il dé-duplique par page (une page déjà vue n'est pas re-déclenchée) → toute **reprise** (relance d'une tâche Blocked→To Do, correction) est perdue. Deux options fiables :
1. **Chemin « Added » uniquement** : acceptable quand chaque déclenchement est une **page neuve** (ex. un nouveau pack de copy). Added est fiable.
2. **Scan planifié + requête d'état** (patron recommandé pour les reprises) : `Schedule Trigger (1 min) → getAll → filtre d'éligibilité (statut + type + relation)`. Reprend tout item éligible qu'il soit créé, modifié **ou relancé**. Idempotence par **mutation d'état** (l'item quitte l'état « à traiter » en fin de run). Traiter **un item par run** (l'aval prend le premier éligible) suffit et évite le recouvrement tant que le run < intervalle.

## 5. Résoudre les RELATIONS en valeurs (leçon centrale)
Quand un agent (LLM) doit **rédiger** à partir de données liées (ex. line-up), il lui faut les **noms**, pas les IDs. Une empreinte d'IDs sert à **détecter un changement**, jamais à écrire. Patron : charger le référentiel lié (`getAll` de la DB cible) une fois, construire une **map id→nom**, résoudre la relation en libellés, injecter ces libellés dans le brief. Sinon l'agent reçoit « non renseigné » et, s'il est bien cadré, **refuse d'inventer** (bon signal) → blocage.

## 6. Difficultés rencontrées (génériques)
1. **Schéma d'API tierce mal deviné** (champ/valeur d'énumération : ex. `initialType` vs `type`) → 400.
2. **Secret d'auth mal saisi** (guillemets, caractère manquant) → 4xx trompeur (ressemble à un problème de permission).
3. **Lecture qui masque des champs** selon le paramètre `fields` → vérifier le bon masque avant de conclure « c'est vide ».
4. **Trigger « Updated » non fiable** → reprises perdues.
5. **Données dans une relation, pas en texte** → agent aval privé de l'info → blocage en cascade.
6. **Attribut envoyé mais non persisté** côté service (comportement API) → ne pas confondre avec un bug d'écriture.

## 7. Solutions implémentées (génériques)
1. Lire le **schéma/méthode réel** de l'API (docs) avant d'envoyer un body ; ne jamais inventer un body.
2. **Secrets saisis par le propriétaire** uniquement ; en cas de 4xx, suspecter d'abord la saisie du secret.
3. Choisir explicitement le **masque de lecture** (`fields=…`) et le documenter.
4. **Scan planifié** pour toute reprise ; « Added » réservé aux vrais events de création.
5. **Résolution relation→noms** systématique côté brief d'agent.
6. Isoler les **comportements API** (persistance partielle) en résidus tracés, pas en bugs fantômes.

## 8. Lessons learned (génériques)
- **Le garde-fou des modifications, c'est le statut** de l'entité (gel au-delà d'un état), pas une date.
- **Link-back = idempotence gratuite** : réécrire l'ID externe sur l'entité rend « create vs update » trivial.
- **Draft-only + gate humain** : l'automate prépare, l'humain publie. Jamais Done/Approved par l'automate.
- **Un refus propre d'un agent vaut mieux qu'une hallucination** : cadrer l'agent pour bloquer + donner la raison.
- **Fiabilité > élégance du trigger** : préférer un scan « bête et fiable » à un trigger « intelligent mais capricieux ».
- **Traiter un item par run** simplifie l'idempotence et borne la charge, au prix d'une latence d'une période — acceptable pour des volumes faibles.

## 9. Réutilisation / paramètres
- Remplacer : service tiers + endpoints (create/update/get) + credential ; attribut de variante ; champs d'entrée ; noms des champs link-back ; DB de tâches et libellé de la tâche pilote ; référentiel de relation à résoudre.
- Constantes AFTRSN : Error WF Sentinel `xzaH0uWy0idKVphF` ; cred Notion `sCsj206WVkNxO0Jl` ; nodes en anglais, contenu EN, doc FR, « · » réservé à la tagline.
