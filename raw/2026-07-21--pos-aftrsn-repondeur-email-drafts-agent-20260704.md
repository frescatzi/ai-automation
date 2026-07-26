---
type: raw
title: "POS-AFTRSN_Repondeur-Email-Drafts-Agent_20260704"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/pos-aftrsn_repondeur-email-drafts-agent_20260704.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# POS AFTRSN — Répondeur email à brouillons via agent (Email-Responder)

Procédure vérifiée le 04.07.2026 — construction, test et publication du processus `AFTRSN-Email-Responder` (`H7fLBXfTAHwfdYts`). Version générique : `POS-GENERIQUE_Repondeur-Email-Drafts-Agent_20260704.md`.

## 1. Prérequis AFTRSN

- Credential **Gmail OAuth2** créée par Karter (même client OAuth que « HELLO AFTRN Google Drive », connexion `hello@aftersunpeople.com`) — **API Gmail activée** dans le projet Google Cloud `909839129866`.
- Agent **AFTRSN-Secretary** (`FtPFOKvi2t18DFb1`) : trigger `Execute Workflow`, contrat `{query}`, lit `$json.query || $json.chatInput`.
- **LuminaOsBot - Telegram** (credential `5kTYhDVm2UDpSDIE`), chat Karter `776345147`.
- **LUMINA-MEMORY-WRITE** (`zu4jfZbmDz8trQLl`), contrat `{brand, title, content, collection, knowledge_type, source, source_ref}`.

## 2. Chaîne construite (10 nodes, dossier `AFTRSN-PROCESS-02-Email-Responder`)

Schedule horaire → Gmail getAll (`in:inbox is:unread newer_than:2d -from:no-reply -from:noreply -category:promotions -category:social -category:updates`, max 20, simplify off) → Code « Filtrer le bruit » (exclusions par expéditeur, extraction `from.text`/`subject`/`text||html`, corps vide rejeté) → Execute Workflow Secretary (prompt à sortie balisée `RESUME_EMAIL:` / `RESUME_DRAFT:` / `---DRAFT---`, règles : langue de l'email, ton AFTER SUN PEOPLE, contexte venue/artiste/général, questions plutôt qu'inventions, **jamais d'engagement ferme**, signature AFTER SUN PEOPLE) → Code « Découper la réponse » (parse les 3 parties, repli sur extraits) → Gmail Create Draft (réponse dans le fil : `options.threadId` + `options.sendTo`) → Gmail markAsRead (idempotence) → Telegram par email (format Karter : 📬 Nouveau draft à relire / 👤 De / ✉️ Objet / 📥 Reçu / ✍️ Draft — **aucun code technique, aucune mention de domaine**) → Code « Compter » → Memoire Save Episode (`aftrsn_memory`, `episodic`) → Set Resume `{status, drafts}`.

Comportement à vide : 0 email → arrêt silencieux après le node Gmail (ni notif ni épisode) — voulu.

## 3. Difficultés rencontrées

1. Node Gmail en `403 Forbidden` malgré la credential connectée.
2. Le décodeur MIME écrit à l'aveugle ne trouvait rien : le premier test filtrait 100 % des emails.
3. Premier email de test rejeté (corps de 1 caractère) — semblait être un bug du filtre.
4. La notification Telegram affichait encore « W-5 » après correction : le PATCH API avait été écrasé par l'autosave de l'éditeur ouvert.
5. chat_id Telegram introuvable (l'API bots n'accepte pas les @username en privé) ; email de test remis « non lu » pas revu immédiatement par la recherche `is:unread`.

## 4. Solutions implémentées

1. Activation de l'API Gmail dans la console Google Cloud du projet du client OAuth (~2 min de propagation).
2. Inspection de la **vraie** sortie d'exécution : le node Gmail (simplify off) renvoie des champs déjà parsés (`text`, `html`, `subject`, `from.text`) — filtre réécrit sur ce format.
3. Diagnostic par les données (textLen=1) : le filtre était correct, l'email était vide — nouveau test avec contenu réaliste.
4. Ré-application du patch puis **rechargement de l'éditeur avant exécution** ; vérification du texte dans l'état serveur ET dans l'état éditeur.
5. chat_id obtenu par Karter via **@rawdatabot** (astuce : l'utilisateur lui envoie un message, le bot renvoie le JSON avec `chat.id`) ; nouvel email de test plutôt que d'attendre l'index Gmail.

## 5. Leçons apprises

1. **Vérifier les prérequis d'API cloud (activation) avant de déboguer le workflow** — un 403 avec credential valide = API désactivée, pas un bug.
2. **Ne jamais coder un parseur contre un format supposé** : exécuter une fois, regarder la vraie sortie, coder contre elle.
3. Quand un filtre rejette tout, regarder **les données** avant le code — le test peut être mauvais, pas le filtre.
4. Après un PATCH API, l'éditeur ouvert est un écrivain concurrent : recharger avant toute action UI (leçon jumelle du POS découpage).
5. Les retours utilisateur sur le **format des notifications** sont des exigences produit à part entière : pas de jargon technique dans ce qui est adressé à l'humain (règle Karter permanente).
