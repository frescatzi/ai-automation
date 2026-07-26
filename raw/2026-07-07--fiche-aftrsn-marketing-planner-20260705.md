---
type: raw
title: "Fiche_AFTRSN-Marketing-Planner_20260705"
source_url: "drive:1kL7NO_GFkKUskEyM6ArSKfp3AhUg49pi"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche technique — AFTRSN-Marketing-Planner (W-4)

**Statut :** 🟢 **v1 + P3 Meta Ads PUBLIÉ** le 05.07.2026 (soir). Contenu (calendrier IG + copies + newsletter) **et campagne Meta en PAUSE** intégrés et testés de bout en bout. **P4 Wix = action de la couche agent (connecteur, sans clé), hors W-4** — voir §7.
**Workflow :** `AFTRSN-Marketing-Planner` — id `4qcia6aWt9ooQrle` — actif.
**Déclenchement :** à la demande, par édition (Execute Workflow), contrat d'entrée `{ edition }` (nom exact de l'édition, ex. `EP5`). Le lancement de campagne reste une décision humaine (cohérent avec P2 humain).
**Sortie :** `{ status, edition, posts_created, newsletter_created }`.

---

## 1 · Ce que fait le workflow (cœur v1)

À partir du nom d'une édition, il génère un **calendrier de contenu Instagram ~3 semaines** + les **copies prêtes à publier** + un **draft de newsletter**, le tout en **brouillon** dans Notion, et notifie sur Telegram.

Chaîne : `Trigger {edition}` → **lecture de la fiche édition** (Notion) → **construction du brief + calendrier** (11 créneaux) → garde `Édition trouvée ?` → **idempotence** (`Vérifier existant`) → garde `Déjà généré ?` → **rédaction par l'agent `AFTRSN-Marketing`** → **parsing JSON** → **création des pages** dans Content & Social → **comptage** → **Telegram** → **mémoire** → **résumé**.

### Calendrier (11 créneaux, relatifs à la date d'édition)
| slot | J- | Type | Thème |
|---|---|---|---|
| teaser | 21 | Teaser | Teaser d'annonce (mystère, « from day to night ») |
| announce | 18 | Post | Annonce officielle (date, ville, concept) |
| lineup1 | 15 | Post | Révélation lineup 1 |
| venue | 12 | Reel | Lieu & ambiance |
| lineup2 | 10 | Post | Révélation lineup 2 / résidents |
| rsvp | 8 | Post | Appel à réserver / billetterie |
| cd7 | 7 | Story | Compte à rebours J-7 |
| reminder | 4 | Post | Rappel programme + horaires day-to-night |
| cd2 | 2 | Story | Compte à rebours J-2 |
| tomorrow | 1 | Reel | C'est demain |
| today | 0 | Story | C'est aujourd'hui (doors) |

Si l'édition n'a pas de date, le calendrier se cale sur aujourd'hui + 21 j.

---

## 2 · Décisions actées (DA-008, tranchée par Karter le 05.07)
- **Périmètre v1 :** la proposition complète — calendrier + copies Instagram (agent Marketing) + draft newsletter + Telegram.
- **Newsletter → fiche Notion** (pas de brouillon Gmail) : page dans **Content & Social** (Channel `Other`, Owner/Agent `Comms`, Type `Post`).
- **Meta Ads (P3) + Wix (P4) : à intégrer cette session dès que les credentials sont fournis** (v1.1).

---

## 3 · IDs & ressources (canoniques — ⚠️ doublons de bases, toujours By ID)
| Ressource | ID |
|---|---|
| Workflow W-4 | `4qcia6aWt9ooQrle` |
| Agent rédacteur | `AFTRSN-Marketing` `wyUEojwZjSKFRaRL` (input `query`, sortie `output`) |
| Base **Editions** (lecture) | `dad7f826-891f-4c29-80e4-183c59bac8b0` |
| Base **Content & Social** (écriture) | `709b816f-e2f8-4dbf-90ee-597803a11158` |
| Base Marketing & KPI (canon, non utilisée en v1) | `63bb41f4-33d1-452e-98cc-51cf3253682b` |
| Credential Notion | `xsBRVxaZ6hhVJWIx` (« Notion account ») |
| Mémoire (LUMINA-MEMORY-WRITE) | `zu4jfZbmDz8trQLl` — collection `episodic`, source_ref `AFTRSN-Marketing-Planner` |
| Telegram | cred `5kTYhDVm2UDpSDIE`, chat `776345147` |
| Site Wix AFTER SUN PEOPLE (pour P4) | `c895f7a6-9708-4fe4-a85b-55add4f393eb` |

### Schéma Content & Social (709b816f)
Title (title) · Channel (select: Instagram/TikTok/Other) · Type (select: Post/Story/Reel/Teaser) · Status (select: Idea/Draft/Needs fix/Brand-check/Approved/Scheduled/Published) · Owner/Agent (select: Supervisor/Marketing/Social/Comms/Brand-check/Founder assistant) · Edition (relation) · Asset link (url) · **Publish date (date) — AJOUTÉ le 05.07 pour W-4**.
> La base n'avait **ni date ni champ de copie**. La copie (caption + hashtags) est écrite dans le **corps de la page** (blocs paragraphe), pas dans une propriété.

---

## 4 · Choix d'implémentation notables
- **API Notion brute via HTTP Request** (pas le node Notion natif) pour lecture et écriture → évite le piège des clés simplifiées `property_*` snake_case et donne un contrôle total (relation, date, blocs enfants). Credential injecté en `predefinedCredentialType: notionApi` + header `Notion-Version: 2022-06-28`.
- **Idempotence** : avant création, requête Content & Social filtrée `Edition relation contains <edition_page_id>`. Si des pages existent déjà → branche `Telegram - Deja fait`, **aucune création** (protège des doubles runs). Un re-run volontaire nécessite de vider/déplacer les pages existantes.
- **Draft-only** : tous les posts et la newsletter sont en `Status = Draft`. Rien n'est publié automatiquement.
- **Parsing robuste** : l'agent est prié de répondre en **JSON strict** ; le node parseur retire d'éventuelles fences, isole le premier `{`…dernier `}`, et lève une erreur explicite si non parsable.

---

## 5 · Difficultés rencontrées → solutions → leçons (session 05.07 soir)
| # | Difficulté | Solution | Leçon |
|---|---|---|---|
| 1 | Node HTTP Notion en erreur « invalid syntax » avec `jsonBody = {{ JSON.stringify({...objet littéral...}) }}` | Remplacer par du **JSON littéral avec valeur interpolée** : `={"filter":{"property":"Edition","title":{"equals":"{{ $json.edition }}"}}}` | n8n confond l'objet littéral `{` juste après `{{` ; ne jamais mettre un objet littéral dans une expression de jsonBody — préférer le JSON en clair avec `{{ champ }}` dans les valeurs |
| 2 | Content & Social sans champ date ni champ copie | Ajout d'un champ `Publish date` (PATCH DB Notion) ; copie dans le **corps** de la page | Toujours lire le **schéma réel** de la base cible avant d'écrire ; un calendrier a besoin d'une date, l'ajouter explicitement |
| 3 | Lecture des prompts d'agent / du `jsCode` des nodes bloquée par le filtre « sensitive » du navigateur agentique | Contrat reconstruit depuis les nodes lisibles + test en direct du contrat (`query` → `output`) | Le filtre est un faux positif récurrent ; le contrat testé empiriquement supplée la lecture du code |
| 4 | La recherche Notion renvoie **plusieurs bases de même nom** (doublons) | La **1re occurrence** de chaque base correspond aux IDs des workflows publiés (canonique) ; toujours **By ID** | Ne jamais se fier au nom : vérifier l'ID contre celui qu'utilisent les workflows en prod |
| 5 | Injecter du code JS (avec accents, `—`, `•`, regex à backticks) via l'API navigateur sans casser l'échappement | Code écrit en fichiers puis **injecté en base64** (`atob` + `TextDecoder` UTF-8) | Pour tout code non trivial poussé par API, passer par base64 élimine les problèmes de quotes/backticks/backslashes/UTF-8 |
| 6 | Token System User Meta : « Aucune autorisation disponible — affectez un rôle d'application à l'utilisateur système » (même app publiée) | Paramètres → **Applications → LUMINA OS → Affecter des personnes → sélectionner l'utilisateur système → activer « Gérer l'app » (Contrôle total)** ; puis regénérer le token | Un token System User n'expose les permissions que si l'utilisateur système a un **rôle dans l'app** ; ni la publication de l'app ni l'assignation du compte pub ne suffisent |
| 7 | Création de campagne Meta : erreur 100 « Invalid parameter » générique | Lire le corps FB réel (`fullResponse`+`neverError`) → champ manquant `is_adset_budget_sharing_enabled` ; l'ajouter (`false`) | Toujours capturer la réponse FB complète pour voir `error_user_title` ; certains comptes exigent `is_adset_budget_sharing_enabled` quand la campagne n'utilise pas de budget CBO |

---

## 6 · Tests réalisés (05.07 soir)
- **Chemin nominal** : édition `ZZTEST-Marketing-Planner` (01.08.2026) → 11 posts + 1 newsletter générés, **12 pages Content & Social** créées (toutes avec Publish date + relation édition + Draft ; Owner Marketing×11, Comms×1 ; répartition Post 6 / Story 3 / Reel 2 / Teaser 1). Voix bilingue DE/EN conforme à la brand voice.
- **Idempotence** : 2e run → branche « Déjà généré », **0 création**, agent non appelé.
- **Édition introuvable** : payload bidon → branche `Erreur - introuvable`.

---

## 7 · P3 Meta Ads — CONSTRUIT ✅ · P4 Wix — couche agent

### P3 Meta Ads (dans W-4) — 🟢 fait le 05.07 au soir
Node **`Meta - Campagne (PAUSE)`** inséré entre `Compter` et `Telegram` (onError = continue → un échec Meta ne bloque jamais le contenu). Crée une **campagne en PAUSE** par édition : `POST https://graph.facebook.com/v21.0/act_1927939831244305/campaigns`, credential n8n **`facebookGraphApi`** `3fAIGHeem6ebftQ3`. Jamais publiée — revue humaine dans Ads Manager. Le Telegram et la mémoire indiquent si la campagne a été créée.
- Corps : `{ name:"AFTER SUN — {edition} — {date}", objective:"OUTCOME_TRAFFIC", status:"PAUSED", special_ad_categories:[], is_adset_budget_sharing_enabled:false }`.
- IDs : **Ad Account `act_1927939831244305`** (ADS by After Sun PEOPLE) · **Page `694773763721517`** (After Sun) · Instagram non lié (optionnel).
- Adsets / creatives / budget = **restent manuels** dans Ads Manager (v1 crée la coquille campagne, prête à recevoir les adsets). Une extension future pourra créer l'adset (ciblage Zürich) + creative depuis un post.

**Mise en place Meta (une fois, faite le 05.07) :** app Meta **LUMINA OS** (id `992613030068111`, use case *Marketing API*, **publiée/Live**, portefeuille After Sun, catégorie Entreprises & Pages, URLs privacy/terms `aftersunpeople.com`) → utilisateur système **LUMINA HERMES N8N** (`61591950411009`) avec le **compte pub (gérer les campagnes)** + la **Page** assignés → **rôle « Gérer l'app » donné à l'utilisateur système sur LUMINA OS** (Paramètres → Applications → LUMINA OS → *Affecter des personnes* → activer *Gérer l'app*) → **token System User « Jamais »** → credential n8n `facebookGraphApi`. Karter saisit le token lui-même (secret).

### P4 Wix — action de la couche agent (pas dans W-4)
La page événement Wix se crée via le **connecteur Wix déjà autorisé** (module Events V3, `CreateEvent` avec `draft:true`, `registration.initialType:"RSVP"` pour un événement sans billet), **sans aucune clé API**. n8n ne pouvant pas appeler le connecteur, cette action est portée par la **couche agent (Hermès / moi)** au lancement de campagne — cohérent avec « Maestro délègue, Hermès exécute — Wix ». Site AFTER SUN PEOPLE = `c895f7a6-9708-4fe4-a85b-55add4f393eb`. Brouillon de démo créé le 05.07 : « AFTER SUN — Zürich · 08.08.2026 » (`b5cb69a2-c569-42b2-8126-d017b38f0e42`, DRAFT).

---

## 8 · Nettoyages
- **Données de test déjà nettoyées par l'agent le 05.07** : 3 éditions de test (`ZZTEST-Marketing-Planner`, `ZZTEST-MP2`, `ZZTEST-MP3`) + **36 pages de test Content & Social** → **archivées** (corbeille Notion, réversibles) ; 2 campagnes Meta de test → **supprimées** (il ne reste que les 3 vraies campagnes de Karter). Content & Social = 0 page active vérifiée.
- **Reste (Karter)** : workflow utilitaire jetable `ZZZ_20260705_tmp_w4_probe` (`Yk8t5R6WQxbr6tmB`, non publié) → à supprimer (DA-009). Brouillon Wix de démo « AFTER SUN — Zürich · 08.08.2026 » (`b5cb69a2-…`, DRAFT) : à garder (c'est la vraie page 08.08, horaire 15h–22h à ajuster) ou supprimer si tu préfères repartir de zéro.
