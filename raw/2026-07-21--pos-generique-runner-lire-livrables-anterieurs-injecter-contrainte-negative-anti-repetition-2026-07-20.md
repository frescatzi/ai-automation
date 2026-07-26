---
type: raw
title: "POS-GENERIQUE_runner-lire-livrables-anterieurs-injecter-contrainte-negative-anti-repetition_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-generique_runner-lire-livrables-anterieurs-injecter-contrainte-negative-anti-repetition_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-GÉNÉRIQUE · Faire diverger un agent générateur multi-passes : lire les livrables antérieurs de la même entité et injecter une contrainte négative

Date : 2026-07-20 · Type : procédure réutilisable (anti-répétition inter-passes)
Origine : chantier AFTRSN, anti-répétition inter-vagues du M2 Copy Runner. Généralisé pour tout moteur où un agent rédige plusieurs passes (vagues, versions, épisodes) sur une même entité.

## 1. Quand l'appliquer
Un agent LLM produit plusieurs livrables successifs pour une même entité (une édition, un client, un produit, une campagne) et chaque passe risque de recycler l'angle/le ton/les phrases de la précédente. On veut de la CONTINUITÉ de marque mais de la DIVERGENCE d'angle. L'agent, seul, ne « voit » pas ses passes antérieures : il faut les lui rappeler explicitement, en négatif.

## 2. Principe
Trois conditions rendent le pattern possible :
1. Chaque passe stocke un marqueur d'angle réutilisable (un champ court « angle » ou « thème »), relié à l'entité et étiqueté par passe (vague/version).
2. Le runner qui prépare le brief peut relire les marqueurs frères de la même entité.
3. Le brief est le canal d'injection : on ajoute une section « déjà utilisé, ne pas réutiliser » AVANT que l'agent rédige.

La contrainte est NÉGATIVE (interdire la répétition) et non prescriptive (on n'impose pas le nouvel angle) : l'agent reste créatif, on borne seulement ce qu'il ne doit pas refaire.

## 3. Recette (indépendante de l'outil)

### Étape A · Garantir la persistance du marqueur d'angle
S'assurer que chaque passe écrit son angle dans la base, relié à l'entité, avec l'étiquette de passe. Sans ça, rien à relire. (Dans AFTRSN : champ `Campaign angle` de la base Website Content, relation `Edition`, champ `Wave`.)

### Étape B · Lire les frères de la même entité
Ajouter au runner une lecture de la base des livrables. Réglages recommandés :
- lecture large puis tri en code (ou filtre natif sur la relation d'entité si le volume est élevé) ;
- `continueOnFail` / `alwaysOutputData` : la lecture ne doit JAMAIS bloquer la génération. Si elle échoue, on rédige sans anti-répétition plutôt que de casser la chaîne.
Câbler ce nœud AVANT le nœud qui construit le brief.

### Étape C · Filtrer en code
Dans le constructeur de brief :
- récupérer l'identifiant de l'entité courante et la passe courante depuis le contexte ;
- parcourir les lignes lues ; retenir celles qui (a) appartiennent à l'entité courante ET (b) ont une passe DIFFÉRENTE de la passe courante ET (c) ont un angle non vide ;
- normaliser les identifiants avant comparaison (ex. retirer les tirets des UUID des deux côtés) ;
- envelopper dans un `try/catch` retombant sur liste vide.

### Étape D · Injecter la contrainte négative
Si la liste n'est pas vide, ajouter au brief une section explicite :
> Angles déjà utilisés sur cette entité (NE PAS les réutiliser ; produire un angle nettement distinct et une formulation neuve, aucune phrase recyclée) : … liste …
Placer cette section après les consignes structurelles (CTA, format) et avant la main à l'agent.

### Étape E · Tester par divergence, sans publier
Injecter un angle antérieur réel, demander la passe suivante, vérifier que le nouvel angle est franchement distinct (pas de phrases recyclées) ET que les invariants de forme tiennent (format, séparateurs, dates, CTA). Ne rien publier pendant le test.

## 4. Pièges et garde-fous
- Ne PAS exclure la passe courante = on s'auto-interdirait des angles légitimes ou on créerait des faux positifs. Toujours filtrer `passe ≠ passe courante`.
- Ne pas rendre la lecture bloquante : dégradation gracieuse obligatoire.
- Contrainte négative, pas prescriptive : on interdit la redite, on n'écrit pas l'angle à la place de l'agent.
- Normaliser les identifiants (UUID avec/sans tirets, casse) avant comparaison, sinon la relation ne « matche » jamais et `priorAngles` reste vide silencieusement.
- Volume : lecture large + tri en code convient tant que la base reste petite ; passer à un filtre natif sur la relation si ça grossit.
- Résidu de forme (code fences, préambule) : compter sur le garde-fou de nettoyage du runner ; documenter le résidu, ne pas le laisser bloquer.

## 5. Critère de « fait »
Le pattern est livré quand : le marqueur d'angle est persistant et relié à l'entité ; le runner lit les frères sans bloquer ; le brief injecte la contrainte négative ; un test de divergence passe (nouvel angle distinct + invariants de forme) sans rien publier ; et c'est documenté (POS exact + addendum Bible + mémoire).
