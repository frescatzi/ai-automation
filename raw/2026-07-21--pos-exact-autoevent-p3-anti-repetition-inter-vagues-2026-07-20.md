---
type: raw
title: "POS-EXACT_AutoEvent-P3_anti-repetition-inter-vagues_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_autoevent-p3_anti-repetition-inter-vagues_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · AutoEvent-P3 · Anti-répétition inter-vagues (M2 Copy Runner)

Date : 2026-07-20 · Projet : Automatisation Événementielle (AutoEvent-P3) · Marque : AFTER SUN PEOPLE (AFTRSN)
Statut : LIVRÉ & TESTÉ (test EP05, rien publié) · Portée : workflow M2 Copy Runner `XQYvzTzFK2NLjDZz`, agent Iris `MWUwLi3vu2Ws65lL`
Auteur d'exécution : session Claude « Claude Lessons ». Documentation en français, contenu publié en anglais.

## 1. Objet et problème résolu
Quand Iris rédige la vague 2 d'une édition (Line-up update) après la vague 1 (Line-up reveal), rien ne l'empêchait de réutiliser le même angle et les mêmes phrases : deux vagues qui se ressemblent, campagne perçue comme répétitive. L'anti-répétition inter-vagues fait lire à M2 les angles déjà rédigés pour CETTE édition et les injecte dans le brief d'Iris comme contrainte négative (« ne réutilise aucun de ces angles, produis un angle nettement distinct »).

Pourquoi c'est possible : chaque pack rédigé stocke son `Campaign angle` dans la base Notion « Website Content », relié à l'édition et étiqueté par vague. On peut donc relire les angles frères d'une même édition.

## 2. Pièces modifiées
- Workflow M2 Copy Runner `XQYvzTzFK2NLjDZz` :
  - AJOUT d'un nœud Notion « Read prior website content ».
  - PATCH du nœud « build-query » (Code) : calcul `priorAngles` + injection de la contrainte négative dans le brief.
- Aucune modification du prompt d'Iris (la consigne arrive par le brief M2, canal habituel).
- Aucune écriture Notion supplémentaire (lecture seule).

## 3. Détail exact du build

### 3.1 Nœud « Read prior website content » (Notion, lecture)
- Type : Notion, opération `getAll` (databasePage).
- Base : Website Content, `databaseId = 1a0c078ddd454477b10c8de1d673c16b`.
- `returnAll = true`, `alwaysOutputData = true`, `continueOnFail = true` (jamais bloquant : si Notion échoue, M2 continue sans anti-répétition plutôt que de casser).
- Credential Notion : `sCsj206WVkNxO0Jl`.
- Câblage : `roster` → **Read prior website content** → `build-query`. (Le nœud s'insère entre la lecture du roster et la construction du brief ; nodeCount du workflow = 19 après ajout.)
- Filtrage : volontairement AUCUN filtre Notion (getAll toutes lignes) ; le tri par édition se fait dans le code de build-query (voir 3.2). Acceptable car M2 ne tourne que sur des éditions en préparation (faible volume). À revoir si le volume Website Content explose (ajouter un filtre relation `edition`).

### 3.2 Patch de « build-query » (Code, JS)
Deux insertions.

(a) Après `const wave = ctx.wave || "Announce";`, calcul des angles déjà utilisés sur cette édition (hors vague courante) :
```js
let priorAngles = [];
try {
  const eid = String(ctx.editionId || '').replace(/-/g,'');
  const rows = $('Read prior website content').all();
  for (const it of rows) {
    const j = (it && it.json) ? it.json : {};
    const rel = Array.isArray(j.property_edition) ? j.property_edition : [];
    const belongs = rel.some(function(x){ return String(x).replace(/-/g,'') === eid; });
    const w = String(j.property_wave || '').trim();
    const ang = String(j.property_campaign_angle || '').trim();
    if (belongs && ang && w && w !== wave) priorAngles.push(w + ': ' + ang);
  }
} catch (err) { priorAngles = []; }
```
Notes :
- On normalise les UUID en retirant les tirets (`replace(/-/g,'')`) des deux côtés pour comparer proprement l'ID d'édition du contexte et la relation Notion.
- On EXCLUT la vague courante (`w !== wave`) : on ne veut interdire que les angles des AUTRES vagues.
- `try/catch` : toute anomalie de lecture retombe sur `priorAngles = []` (dégradation silencieuse, jamais de blocage).

(b) Après la ligne CTA `L.push("- " + ctaLine);`, injection de la contrainte négative dans le brief seulement s'il y a des angles antérieurs :
```js
if (priorAngles.length) {
  L.push("");
  L.push("Angles already used on this edition (do NOT reuse them; produce a clearly distinct angle and fresh wording, no recycled phrases):");
  for (const pa of priorAngles) L.push("- " + pa);
}
```

## 4. Test de validation (EP05, rien publié)
Méthode : déclenchement d'Iris via son chat trigger (`mcp__n8n__execute_workflow`, type chat), en injectant l'angle réel du reveal EP05 comme angle antérieur, puis demande de la vague « Line-up update ».

Angle antérieur injecté (reveal EP05) : « EP05 comes home to Zürich with three names who already belong to this city's rhythm : Aïssa, Paqo, Bambona… ».

Résultat : Iris a produit un angle nettement distinct. L'update parle de QUI REJOINT le line-up (expansion, cercle qui s'élargit, nouveau nom local Mila qui entre), pas de qui est déjà là. Aucune phrase recyclée du reveal.
Contrôles de conformité passés en même temps : CTA FREE correct (« Free registration, secure your spot »), séparateurs middot « · », date 26.06.2026 au format DD.MM.YYYY dans la ligne pratique, un seul emoji sur l'Instagram.

Résidu connu (sans impact) : la sortie était encore emballée dans des code fences ```` ```json ````. Le garde-fou de M2 les retire avant écriture Notion ; le livrable n'est pas affecté. Variance résiduelle du modèle, documentée, non bloquante.

## 5. Vérification post-build
- jsCode de build-query relu après patch : les deux insertions présentes, syntaxe correcte.
- Câblage confirmé : roster → Read prior website content → build-query, nodeCount 19.
- Preuve que `priorAngles` se peuple : la page Website Content EP05 (`3a224f2fef0c810091edfdc2c350bd31`) stocke bien « Campaign angle » du reveal, Wave = « Line-up reveal », relation Edition → EP05.

## 6. Invariants respectés
Pas de tiret cadratin · numérotation à partir de 1 · rien publié (test chat trigger n'écrit rien dans Notion) · contrat de sortie Iris 8 champs JSON inchangé · français pour la doc, anglais pour le contenu.

## 7. Suites possibles (basse priorité)
- Ajouter un filtre relation `edition` côté Notion si le volume Website Content grossit (aujourd'hui getAll + tri en code, suffisant).
- Étendre le principe à d'autres runners multi-vagues si besoin (voir POS-GÉNÉRIQUE associé).
