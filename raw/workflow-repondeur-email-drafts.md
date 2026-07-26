---
type: wiki
title: "AFTRSN — Workflow Répondeur email à brouillons (Email-Responder)"
status: draft
publish: none
vault: brands
brand: AFTRSN
sources: [AFTRSN/raw/2026-07-21--pos-aftrsn-repondeur-email-drafts-agent-20260704.md]
related: [[architecture-processus-metier]], [[methode-construction-workflow-n8n]]
updated: 2026-07-21
---

# AFTRSN — Workflow Répondeur email à brouillons (Email-Responder)

Workflow **`AFTRSN-Email-Responder`** (`H7fLBXfTAHwfdYts`), publié le 04.07.2026, dossier `AFTRSN-PROCESS-02-Email-Responder`. Réalise le processus transverse **« Réponse email »** de [[architecture-processus-metier]] : lecture Gmail horaire → brouillons → notification Telegram → mémoire ; **jamais d'envoi automatique**.

## Prérequis

- Credential **Gmail OAuth2** (client OAuth partagé avec « HELLO AFTRN Google Drive », compte `hello@aftersunpeople.com`) — API Gmail activée dans le projet Google Cloud `909839129866`.
- Agent **AFTRSN-Secretary** (`FtPFOKvi2t18DFb1`), trigger `Execute Workflow`, contrat `{query}`.
- **LuminaOsBot - Telegram** (`5kTYhDVm2UDpSDIE`), notifications adressées à Karter (`776345147`).
- **LUMINA-MEMORY-WRITE** (`zu4jfZbmDz8trQLl`), appelé en Execute Workflow, contrat `{brand, title, content, collection, knowledge_type, source, source_ref}`.

## Chaîne (10 nodes)

Schedule horaire → **Gmail getAll** (`in:inbox is:unread newer_than:2d -from:no-reply -from:noreply -category:promotions -category:social -category:updates`, max 20, simplify off) → **Filtrer le bruit** (exclusions par expéditeur, extraction `from.text`/`subject`/`text||html`, rejet des corps vides) → **Execute Workflow Secretary** (prompt à sortie balisée `RESUME_EMAIL:` / `RESUME_DRAFT:` / `---DRAFT---` ; règles : répondre dans la langue de l'email, ton AFTER SUN PEOPLE, contexte venue/artiste/général, poser des questions plutôt qu'inventer, **jamais d'engagement ferme**, signature AFTER SUN PEOPLE) → **Découper la réponse** (parse des 3 parties, repli sur extraits si le balisage est incomplet) → **Gmail Create Draft** (dans le fil existant : `options.threadId` + `options.sendTo`) → **Gmail markAsRead** (idempotence) → **Telegram par email** (format humain : 📬 Nouveau draft à relire / 👤 De / ✉️ Objet / 📥 Reçu / ✍️ Draft — aucun code technique, aucune mention de domaine) → **Compter** → **Mémoire Save Episode** (`aftrsn_memory`, `episodic`) → **Set Resume** `{status, drafts}`.

**Comportement à vide (voulu) :** 0 email non lu → arrêt silencieux juste après le node Gmail, sans notification ni épisode mémoire.

## Points d'attention spécifiques à ce build

- Le décodeur du node « Filtrer le bruit » doit être écrit contre la **vraie sortie** du node Gmail en `simplify off` (champs déjà parsés `text`, `html`, `subject`, `from.text`) — pas contre un format MIME supposé (cf. [[methode-construction-workflow-n8n]] §Exécution, données et sorties multi-items).
- Un email de test au corps vide (1 caractère) peut faire croire à un bug de filtre alors que le filtre est correct : diagnostiquer par la donnée (longueur du corps) avant de suspecter le code.
- Un email remis « non lu » pour retest n'est pas forcément repris immédiatement par la recherche `is:unread` de Gmail — préférer un nouvel email de test plutôt qu'attendre l'indexation.

Les autres difficultés rencontrées sur ce build (403 Gmail par API cloud non activée, notification Telegram écrasée par l'autosave de l'éditeur après un PATCH, chat_id Telegram introuvable pour un `@username`) confirment des pièges déjà consignés dans [[methode-construction-workflow-n8n]] et n'apportent rien de nouveau à documenter ici.

## Leçon transverse

Les retours utilisateur sur le **format des notifications** (Telegram, etc.) sont des exigences produit à part entière, pas des détails cosmétiques : pas de jargon technique dans ce qui est adressé à l'humain (règle Karter permanente, déjà appliquée au format de notification ci-dessus).
