---
type: raw
title: "Fiche_AFTRSN-Email-Responder_20260704"
source_url: "drive:105fgnXeHWIHkTT3q4n5bpoY7kg4dUmP6"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# Fiche workflow — AFTRSN-Email-Responder

**Format :** conforme à la Bible 360° LUMINA OS §E.

---

**ID :** `H7fLBXfTAHwfdYts` — **Criticité :** 🟡 (important)
**Statut :** 🟢 **publié le 04.07.2026** (« v1.0 — Email-Responder »), déclenchement horaire actif
**Emplacement :** `Personal / 02-AFTRSN / AFTRSN-02-BUSINESS AUTOMATION / AFTRSN-PROCESS-02-Email-Responder`
**Tags :** 🌞 AFTRSN · 🔄 automation · 🟢 PROD
**Déclencheur :** Schedule — toutes les heures

## Rôle

Processus transverse « Réponse email » : lit les emails non lus de `hello@aftersunpeople.com`, fait rédiger un brouillon de réponse contextualisé par l'agent **AFTRSN-Secretary**, dépose le brouillon dans le fil Gmail, notifie Karter sur Telegram avec un résumé, journalise l'épisode. **Aucun envoi automatique, jamais** — Karter relit et envoie lui-même depuis Gmail (invariant : action critique = validation humaine ; Secretary = draft-only).

## Chaîne (10 nodes)

`Toutes les heures` → `Gmail — Emails non traités` (non-lus, boîte de réception, ≤ 2 jours, requête excluant no-reply/promotions/social/updates, max 20) → `Filtrer le bruit` (Code : exclusions par expéditeur, extraction from/subject/body — champs **déjà parsés** par le node Gmail : `text`, `html`, `from.text` ; corps vide = rejeté) → `Secretary — Rédiger le draft` (Execute Workflow → `AFTRSN-Secretary` `FtPFOKvi2t18DFb1`, input `{query}` ; format de sortie balisé `RESUME_EMAIL:` / `RESUME_DRAFT:` / `---DRAFT---`) → `Découper la réponse` (Code : parse les 3 parties, repli sur extraits si balises absentes) → `Gmail — Créer le brouillon` (draft en réponse dans le fil, destinataire = expéditeur) → `Gmail — Marquer traité` (markAsRead = idempotence) → `Telegram — Notifier` (LuminaOsBot → chat 776345147, **un message par email** : 📬 Nouveau draft à relire / 👤 De / ✉️ Objet / 📥 Reçu / ✍️ Draft — sans codes techniques, règle Karter) → `Compter` → `Memoire - Save Episode` (→ `LUMINA-MEMORY-WRITE` `zu4jfZbmDz8trQLl`, collection `episodic`) → `Resume` (`{status, drafts}`).

**Comportement à vide :** 0 email non lu → la chaîne s'arrête après le node Gmail, sans notification ni épisode mémoire (voulu).

## Règles du prompt Secretary

Langue de l'email reçu ; ton AFTER SUN PEOPLE (« Good People. Great Vibes. Beautiful Energy. ») ; contexte auto-détecté (venue / artiste-booking / général) ; questions plutôt qu'inventions si infos manquantes ; **jamais d'engagement ferme, de dépense ou d'accord contractuel** ; signature « AFTER SUN PEOPLE ».

## Dépendances & credentials

- **Amont :** Gmail hello@ (credential `Gmail account`, API Gmail activée le 04.07 dans le projet Google Cloud `909839129866`)
- **Aval :** AFTRSN-Secretary (agent, OpenRouter + mémoire Postgres), LuminaOsBot - Telegram (`5kTYhDVm2UDpSDIE`), LUMINA-MEMORY-WRITE
- Aucun credential saisi par l'agent — Gmail créé et connecté par Karter.

## Construit / testé / publié le 04.07.2026

Construit via l'API REST interne n8n. Bugs rencontrés et corrigés pendant le test : API Gmail non activée côté Google Cloud (403) ; décodeur MIME inutile (le node Gmail « simplify off » renvoie des champs déjà parsés `text`/`html`/`from`) ; premier email de test à corps vide correctement rejeté par le filtre ; patch API écrasé par l'autosave de l'éditeur ouvert (leçon : recharger l'éditeur après tout PATCH API). Test final : 2 emails → 2 drafts + 2 notifications au format validé par Karter.

## ⚠️ Ne pas toucher sans précaution

- L'idempotence repose sur `is:unread` + markAsRead : un email lu manuellement par Karter avant le run horaire ne sera pas traité (comportement assumé — laisser non lu ce qui doit être traité).
- Le format balisé de la sortie Secretary est parsé par `Découper la réponse` — si le prompt change, garder les balises `RESUME_EMAIL:`, `RESUME_DRAFT:`, `---DRAFT---`.
- Notification Telegram : pas de codes techniques (« W-5 », etc.) ni de mention du domaine — règle explicite Karter du 04.07.2026.
