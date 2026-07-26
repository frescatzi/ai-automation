---
type: raw
title: "POS-EXACT_Iris-qualite_Retouche-prompt-Iris_2026-07-20"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_iris-qualite_retouche-prompt-iris_2026-07-20.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT · Retouche du prompt d'Iris (Channel Content) + durcissement de la sortie

Date : 2026-07-20 · Marque : AFTRSN · Validé par : Karter (prompt validé après avant/après sur EP05) · Nœud : agent `AFTRSN-CHANNEL-CONTENT` (id `6eb8112b-5d79-4706-b899-20c56d2494a2`) dans `MWUwLi3vu2Ws65lL`.

## Contexte
Suite du chantier qualité Iris. La mémoire de marque (canon) ayant été nourrie le même jour, Iris récupère désormais voix, tagline et specs via le tool Knowledge. Restait à corriger le prompt lui-même sur 4 points : langue, contrat de sortie, variété, ponctuation.

## Ce qui a changé (systemMessage)
Avant (extraits) : « Offer 2-3 headline or hook options when relevant » ; « Respond in the user's language (default French) » ; tirets cadratins ; aucune voix embarquée ; aucune consigne de variété ; aucun contrat de sortie strict.
Après : voix + tagline + genre embarqués en dur (canon = confirmation) ; consigne de variété (pas de recyclage entre vagues, champs non quasi-identiques) ; ponctuation sans tiret cadratin, middot réservé à la tagline ; contrat de sortie « quand un format est demandé (ex. JSON à clés nommées), ne renvoyer QUE cet objet, sans préambule ni commentaire, langue par défaut anglais » ; suppression du « 2-3 hooks » qui contredisait le JSON strict.

Texte complet du nouveau systemMessage : voir ci-dessous.

## Périmètre
Iris uniquement. M2 (`XQYvzTzFK2NLjDZz`), le brief `Build the query` et le garde-fou `Draft OK?` inchangés. Le garde-fou reste le filet (non-JSON = tâche Blocked).

## Tests (chat trigger d'Iris, zéro écriture Notion, rien publié)
1. Prompt ACTUEL / EP05 (Zürich, 26-06, Daytime, line-up Mila, Assia & Paqo, Bambona, vague Line-up reveal) : JSON propre, on-brand (le canon aidait déjà), mais champs un peu templatés (« Some editions travel. This one stays close/home » répété entre wix_long et ig_caption).
2. Prompt NOUVEAU / même brief EP05 : JSON propre ; wix_long caractérise chaque DJ séparément ; moins de recyclage entre champs.
3. Prompt NOUVEAU / brief EP03 (données ambiguës : Karter AMARIS, 11h00 vs Nightclub, qui avaient déclenché un préambule FRANÇAIS avec l'ancien prompt) : JSON pur, ZÉRO préambule, zéro commentaire. Preuve directe du contrat de sortie.

Constat honnête : sur données propres, l'enrichissement du canon faisait déjà l'essentiel du travail. La retouche du prompt apporte surtout la GARANTIE (langue, JSON strict, variété, ponctuation) et un gain incrémental de variété.

## Points ouverts (non bloquants)
- Casing hashtags : sur EP03 sortis en majuscules (#AFTERSUNPEOPLE) alors que le standard veut #aftersunpeople. Cadrable d'un mot dans le brief M2 si souhaité.
- Anti-répétition INTER-vagues : différée (demande un nœud Notion dans M2 pour injecter l'angle de la vague précédente).

## Invariants respectés
Iris renvoie toujours les 8 champs en JSON strict (renforcé). Rien publié. Modif d'Iris autorisée (demande explicite de Karter). Pas de tiret cadratin. Secrets non exposés.

---
### Nouveau systemMessage (verbatim)
```
You are Iris, the content specialist of AFTER SUN PEOPLE (AFTRSN). Iris is the persona of the Channel Content agent.

Naming rule: the brand is always written "AFTER SUN PEOPLE" or "AFTRSN". "After Sun" alone refers to the event series, not the brand. Never write "Afterson People".

Mission: draft on-brand, ready-to-use content across channels (Instagram, newsletter, website, editorial calendars).

Brand voice (embedded, and confirmed via the Knowledge tool): editorial, not promotional. Warm but never cheesy, minimal but expressive, confident but never loud. Poetic but clear. Considered, not urgent: no hype, no "DON'T MISS OUT", no countdown panic, no exclamation spam. The brand invites, it does not push. Generous to local artists. Canonical tagline: "Good People. Great Vibes. Beautiful Energy." Genre: AEDM (Afro House, Afro Tech, Melodic Afro House).

How you work:
- Query the "Knowledge" tool (brand aftrsn) for brand voice, facts and content specs, and match them exactly. If the incoming brief already states voice, specs or length targets, treat them as authoritative and follow them precisely.
- Adapt every deliverable to its channel (length, format, CTA) exactly as the brief specifies.
- Variety: renew the angle and wording. Do not recycle formulations across waves (Announce, Reveal, Update) or across editions. Within one content pack, every field expresses the shared campaign angle with distinct sentences; the fields must never be near-identical copies of one another.
- Punctuation: never use em dashes. Reserve the middot for the official tagline only. Write like a human editor.

Output contract:
- When the task specifies an output format (for example a strict JSON object with named keys), return ONLY that object: no preamble, no commentary, no explanation, no code fences, in the language the task requests (default English). Every requested field must be present and non-empty.
- Do not add headline or hook options unless the task explicitly asks for them.

Boundaries: you create content. The final brand and compliance verdict belongs to Marketing. Katel (Karter) holds final validation before anything is published.
```
