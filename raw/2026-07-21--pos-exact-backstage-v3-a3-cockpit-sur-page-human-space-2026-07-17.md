---
type: raw
title: "POS-EXACT_Backstage-v3_A3-cockpit-sur-page-human-space_2026-07-17"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-exact_backstage-v3_a3-cockpit-sur-page-human-space_2026-07-17.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS-EXACT — Backstage v3 · Amendement 3 « Cockpit affiché SUR la page Human Space » (dissolution de la sous-page)

Date : 17.07.2026 · Marque : AFTER SUN PEOPLE (AFTRSN) · Espace : Notion « The Backstage » (teamspace AFTER SUN PEOPLE - HQ)
Statut : **FAIT & VALIDÉ par Karter le 17.07.2026** (« tout est bon pour moi »)
Référence : `SPEC-BUILD_Notion-Backstage-v3_2026-07-16.md` (Amendements en tête) · POS A2 du même jour (la LOI sidebar y est documentée)

## Objectif

Karter veut retrouver l'expérience « cockpit sur la page d'accueil », mais transposée à Human Space : en ENTRANT dans Human Space, on voit directement les datas du cockpit (base Tasks + 4 vues), sans clic supplémentaire. La sous-page 🔭 Cockpit (A2) devient inutile → dissoute. La sidebar sous Backstage reste minimale (les vues se rangent sous Human Space — voulu).

## Contrainte majeure de la session

**Le connecteur Notion est resté mort malgré la réautorisation par Karter** (« Server not connected » côté session, alors que claude.ai affichait Connecté). Toute l'opération a été faite **au NAVIGATEUR (Claude in Chrome)** — le Chrome de Karter est loggé Notion. C'est le POS de référence pour ce mode dégradé.

## Marche à suivre exacte (100 % navigateur)

1. **Ouvrir la page 🔭 Cockpit** (`3a024f2fef0c811880abc44d93238ca6`).
2. **Pour chacun des 7 blocs, dans l'ordre d'affichage souhaité à l'arrivée** (base Tasks → ⏭️ Up Next → 🎪 Upcoming Editions → 📣 Publishing Pipeline → titre `## ✅ Awaiting validation` → sa description → vue To Validate) : survoler le bloc → cliquer la poignée **⋮⋮** → **Move to** → taper « Human » → **choisir « Human Space — AFTER SUN PEOPLE - HQ / The Backstage »** (⚠️ le doublon LUMINA est juste en dessous). Chaque bloc arrive EN FIN de la page destination → l'ordre des moves = l'ordre final.
3. **Corbeiller la page Cockpit vide** : ••• (haut droite de la page) → **Move to Trash**. Son entrée dans la liste 🚪 Rooms de Human Space disparaît automatiquement.
4. **Insérer le titre de section sur Human Space** : survoler le bloc Tasks (premier bloc arrivé) → **alt+clic sur « + »** (= insérer AU-DESSUS) → Escape (fermer le slash-menu) → taper `## 🔭 Cockpit` (le `## ` se convertit en H2) → Entrée → taper l'intro → `cmd+a` puis `cmd+i` (italique).
5. **Vérifier** : page Human Space = intro → 🚪 Rooms (Operations / Knowledge / Team) → ## 🔭 Cockpit + intro → base Tasks (board complet) → Up Next (4 tâches, alertes Overdue) → Upcoming Editions (EP6 Live) → Publishing Pipeline → ✅ Awaiting validation + vue To Validate. Sidebar sous Backstage inchangée (Human Space · Agents Space · Archive).
6. **Validation Karter** → maj mémoire + ce POS + spec.

## Fausse alerte traitée dans la foulée

« Content & Social a disparu » → NON : elle avait été renommée « 📣 Publishing Pipeline » lors des manips manuelles du 16.07 (piège rename), réparée plus tôt dans la session (`update-data-source title`) → elle réapparaît sous son vrai nom dans Operations → Marketing & Content. Vérifié visuellement, capture à l'appui. Leçon : après réparation d'un renommage de data source, prévenir l'utilisateur que la base « réapparaît » sous son ancien nom — sinon il croit à une disparition.

## Difficultés rencontrées

- Connecteur Notion : 2ᵉ expiration de jeton en 24 h, et cette fois la réautorisation ne s'est PAS propagée à la session en cours (outils jamais revenus pendant l'opération ; revenus après coup).
- Deux « Human Space » homonymes dans le picker Move to (AFTRSN HQ + LUMINA) — piège des 2 Backstages, encore.
- Slash-menu s'ouvrant automatiquement sur le bloc inséré par « + » (il faut Escape avant de taper).
- Premier caractère parfois avalé quand on tape immédiatement dans le champ de recherche du Move to → vérifier le champ, cmd+a et retaper si besoin.

## Solutions implémentées

- Fallback navigateur complet documenté ci-dessus (Move to bloc par bloc, ordre des moves = ordre final).
- Désambiguïsation systématique par le sous-titre du résultat (« AFTER SUN PEOPLE - HQ / The Backstage »).
- alt+clic sur « + » pour insérer au-dessus d'un bloc existant (évite tout drag).

## Lessons learned

- **Le menu « Move to » d'un bloc est le fallback universel des `move-pages` API** : il déplace databases, linked views, headings et textes, un par un, et l'ordre des moves détermine l'ordre d'arrivée (toujours en fin de page destination).
- **Une réautorisation de connecteur peut ne jamais se propager à la session en cours** : ne pas s'acharner (3 attentes + re-tests suffisent), basculer sur le navigateur.
- **« Sous-page dédiée » vs « section sur la page »** : même contenu, deux navigations. La section sur la page d'entrée = zéro clic pour voir les datas ; la sous-page = page d'entrée plus courte. Le choix appartient à l'utilisateur et il est réversible en ~10 moves.
- Après toute réparation de nom de data source, **annoncer explicitement le retour de l'ancien nom** pour éviter les fausses alertes de disparition.
