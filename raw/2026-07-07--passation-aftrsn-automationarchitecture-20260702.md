---
type: raw
title: "PASSATION_AFTRSN_AutomationArchitecture_20260702"
source_url: "drive:1gXZIYV9VdLcBDQ-gQPHU0KR20vAwHuFb"
captured: 2026-07-07
vault: brands
brand: AFTRSN
immutable: true
---

# PASSATION — AFTERSUN Automation Architecture
**Projet :** AFTRSN-AI-Operating-System  
**Session :** Architecture MVP — Automation workflows n8n  
**Date :** 02.07.2026  
**Destinataire :** Cowork / prochain assistant  
**Statut :** Architecture validée — prêt pour implémentation
---
## Contexte & mission
AFTERSUN est une marque d'événementiel et lifestyle musical opérée en solo par Karter. L'objectif est de construire un système d'automatisation n8n qui gère le cycle de vie complet d'un événement : de la prospection des lieux et artistes jusqu'au closing budgétaire et à la capture de connaissances post-event.
Le prochain événement est le **08.08.2026**. C'est le premier event à automatiser et la deadline cible du MVP.
**Philosophie retenue :** Business architecture first → data model → process landscape → automation → n8n. Jamais l'inverse.
---
## Infrastructure existante (Lumina OS)
Le système s'intègre dans **Lumina OS**, déjà en production. Ne pas reconstruire ce qui existe.
### Composants actifs à réutiliser
| Composant | Rôle | Usage AFTRSN |
|---|---|---|
| `AFTRSNMaestro` | Hub orchestrateur de la marque | Point d'entrée de tous les nouveaux workflows AFTRSN |
| `AFTRSNSecretary` | Rédaction emails, draft-only | W-2 Venue-Coordinator · W-3 Artist-Coordinator · W-5 Email-Responder |
| `AFTRSNMarketing` | Stratégie marketing | W-4 Marketing-Planner |
| `AFTRSNCHANNELCONTENT` | Contenu social + newsletter | W-4 Marketing-Planner |
| `AFTRSNComptable-Finance` | Finance (câblé, non publié — décision Karter attendue) | W-6 Budget-Tracker — à publier |
| `LUMINAHermes-Exec` | Outil Ops (web search, exécution) | W-4 pour Canva |
| `LUMINAAIRouter` | Routing multi-LLM par task_type | Appelé par tous les nouveaux sous-agents |
| `LUMINAMEMORYWRITE/WEBHOOK` | Save Episode dans pgvector | W-7 Knowledge-Capture |
| `LUMINAMEMORYCONSOLIDATION/NIGHTLY` | Consolidation nocturne (cron 03h00) | W-7 automatiquement |
| `LUMINAMCPServer` | Endpoint MCP externe (`search_brand_memory`) | Accessible depuis Claude |
### Banque mémoire AFTRSN
- **Table :** `aftrsn_memory`
- **Collections :** `canon` (templates de référence) · `episodic` (instances d'events) · `insights` (post-event) · `docs` · `skills`
- **Embedding :** `text-embedding-3-small` 1536 dims — **FIGÉ, ne pas toucher**
### LLMs en place
| Modèle | Via | Rôle |
|---|---|---|
| Claude Sonnet (latest) | OpenRouter | Maestro · Marketing · Channel-Content · Comptable · Qualite · Consolidation |
| Gemini Flash (latest) | OpenRouter | Culture-Steward · Experience-Designer |
| GPT-mini (latest) | OpenRouter | Secretary |
| text-embedding-3-small | OpenAI direct | Toute la mémoire — FIGÉ |
| Hermes (self-hosted) | hermes.aftersunpeople.com | Hermes-Exec |
### Credentials n8n existants
`Postgres` · `OpenAI` · `OpenRouter (w8bIvwIxqs1v3S0g)` · `GitHub` · `Notion` · `Google Drive - Live` · `Google Drive - FETCH` · `Google Gemini API` · `Hermes WebUI Custom Auth (LoNa2cG7DqHzzUHv)`
**⚠ Manquant pour ce projet :** Google OAuth (Gmail + Calendar + Sheets) pour le compte AFTRSN — à créer en étape 1.
### Piège technique critique
> Le node Anthropic natif n8n génère une erreur 404. Toujours utiliser **HTTP Request** pour appeler Claude. n8n tourne en mode queue (n8n + n8n-worker).
---
## Architecture métier validée
### Principe fondamental
L'**Event** est l'objet métier central. Tous les autres objets s'y rattachent.
```
Event
├── Venue
├── Artists[]
├── Budget
├── Tasks[]
├── Campaigns[]
├── Contracts[]
├── Invoices[]
├── ContentAssets[]
└── TeamMembers[]
```
### 2 familles de processus
**Business Development** — toujours actif, indépendant des events  
**Event Delivery** — déclenché uniquement quand venue + date sont confirmés
### Objets métier MVP (Niveau 1)
`Event` · `Venue` · `Artist` · `Budget` · `Task` · `Campaign` · `ContentAsset` · `Contract` · `Invoice` · `TeamMember`
Niveau 2 (post-08.08) : `Sponsor` · `Ticket` · `Supplier` · `Partner` · `Customer`
---
## Process landscape
### P-1 · Venue Prospecting & Partnership
`Identify → Outreach → Follow-up auto → Negotiate → Contract → Confirm`  
Trigger de sortie : status = CONFIRMED → déclenche P-3 Event Creation
### P-2 · Artist Relations & Booking
`Scout → Outreach → Follow-up auto → Book → Contract → Brief D-7`
### P-3 · Event Creation & Management
`Trigger (venue confirmé) → Init → Budget → Checklist → Monitor → Closing`
### P-4 · Marketing Campaign
`Content Plan → Announce → Warm-up → Last Push → Post-event`  
Durée cible : 3 semaines de promo avant l'event
### P-5 · Budget Management
Transversal à P-3 — template auto à l'init, alertes seuils, closing report
### P-6 · Knowledge Capture
Post-event — debriefs → `aftrsn_memory` collection `insights` → consolidation nocturne
---
## MVP Workflows n8n — 7 workflows à construire
> Règle : 1 workflow = 1 processus = 1 objectif. Jamais monolithique.
### W-1 · AFTRSN-Event-Creator
**Criticité :** 🔴 Critique  
**Déclencheur :** Manuel via Maestro  
**Objectif :** Créer la fiche event, le Drive folder structuré, l'entrée Calendar et le budget Sheets template en une seule commande  
**Intégration Lumina OS :** Nouveau sous-agent AFTRSN → écrit dans `aftrsn_memory` (collection `episodic`)  
**Outils requis :** Google Drive · Google Calendar · Google Sheets  
**Statut :** À construire — c'est le premier
---
### W-2 · AFTRSN-Venue-Coordinator
**Criticité :** 🔴 Critique  
**Déclencheur :** Manuel / webhook Maestro  
**Objectif :** Rédiger les emails d'outreach venue dans le ton AFTRSN, programmer les follow-ups automatiques J+3/J+7, tracker le statut des venues dans Google Sheets  
**Intégration Lumina OS :** Réutilise Secretary (draft-only) + Hermes-Exec  
**Outils requis :** Gmail AFTRSN · Google Sheets  
**Statut :** À construire — semaine 2
---
### W-3 · AFTRSN-Artist-Coordinator
**Criticité :** 🔴 Critique  
**Déclencheur :** Manuel / Scheduled  
**Objectif :** Outreach booking artistes, follow-ups auto, brief D-7 (set time + rider + logistique)  
**Intégration Lumina OS :** Réutilise Secretary + Hermes-Exec  
**Outils requis :** Gmail AFTRSN · Google Calendar  
**Statut :** À construire — semaine 2
---
### W-4 · AFTRSN-Marketing-Planner
**Criticité :** 🟡 Important  
**Déclencheur :** Trigger automatique quand event = CONFIRMED  
**Objectif :** Générer le content calendar 3 semaines, copywriting Instagram dans le brand voice AFTRSN, drafts newsletter, assets Canva via Hermes  
**Intégration Lumina OS :** Réutilise CHANNEL-CONTENT + Marketing (existants)  
**Outils requis :** Canva MCP · Gmail · Instagram  
**Statut :** À construire — semaine 3
---
### W-5 · AFTRSN-Email-Responder
**Criticité :** 🟡 Important  
**Déclencheur :** Scheduled toutes les heures  
**Objectif :** Lire Gmail AFTRSN, générer des drafts de réponse contextualisés (venue / artiste / général), envoi sur validation Karter uniquement  
**Intégration Lumina OS :** Secretary existant + Gmail MCP connecté  
**Outils requis :** Gmail AFTRSN  
**Statut :** À construire — semaine 1
---
### W-6 · AFTRSN-Budget-Tracker
**Criticité :** 🟡 Important  
**Déclencheur :** Manuel / Scheduled  
**Objectif :** Lire le Google Sheets budget, alerter si seuils dépassés, générer un rapport de closing post-event  
**Intégration Lumina OS :** Comptable-Finance (existant, câblé mais non publié — décision à prendre : le publier)  
**Outils requis :** Google Sheets  
**Statut :** À construire — semaine 2
---
### W-7 · AFTRSN-Knowledge-Capture
**Criticité :** ⚪ Utilitaire (post-event)  
**Déclencheur :** Manuel post-event  
**Objectif :** Ingérer les debriefs post-event dans `aftrsn_memory` collection `insights`. La consolidation nocturne LUMINA existante prend le relais automatiquement.  
**Intégration Lumina OS :** LUMINAMEMORYCONSOLIDATION/NIGHTLY (existant)  
**Outils requis :** Google Drive → pgvector  
**Statut :** À construire — post 08.08
---
## Ordre d'implémentation
### Étape 1 — Semaine 1 (02–08 juillet) · Fondations
1. Créer credential Google OAuth dans n8n (Gmail AFTRSN + Drive + Calendar + Sheets)
2. Tester la connexion sur un workflow existant (Secretary de préférence)
3. Construire et tester **W-1** Event-Creator
4. Construire et tester **W-5** Email-Responder
### Étape 2 — Semaine 2 (09–15 juillet) · Coordination
5. Construire **W-2** Venue-Coordinator
6. Construire **W-3** Artist-Coordinator
7. Construire **W-6** Budget-Tracker
8. Décider du statut de `AFTRSNComptable-Finance` (#22) — publier ou documenter la raison
### Étape 3 — Semaine 3 (16–22 juillet) · Marketing
9. Construire **W-4** Marketing-Planner
10. Lancer la campagne 08.08 (annonce officielle Instagram + newsletter)
### Étape 4 — Post-event (août) · Clôture
11. Construire **W-7** Knowledge-Capture
12. Closing budget + rapport financier
---
## Deadline critique — 18 juillet
> **Venue + minimum 2 artistes confirmés avant le 18 juillet.**  
> C'est J-21 par rapport à l'event. En dessous de cette date, la campagne promo est réduite à 2 semaines au lieu de 3. Risque direct sur la fréquentation.
---
## Invariants à respecter (hérités de Lumina OS)
1. **PII jamais au cloud.** `task_type='private'` → bloqué par le Router, quel que soit l'appelant.
2. **Actions critiques = validation humaine.** Dépense, envoi d'email, publication, suppression — toujours gatées. Secretary = draft-only. W-5 envoie uniquement sur approbation Karter.
3. **Embedding FIGÉ.** `text-embedding-3-small` 1536 dims. Ne pas changer — impliquerait de ré-encoder toutes les banques.
4. **Git = source de vérité.** pgvector et Notion sont des dérivés régénérables. Ne jamais les éditer à la main.
5. **n8n fait foi.** En cas d'écart entre doc et workflow live, vérifier le workflow avant d'agir.
6. **Suppressions / DROP = réservés à Karter.** Aucune action destructive (fichiers, tables, workflows).
7. **Ne jamais réactiver un `ZZZ_` sans revue complète.**
8. **Piège `jsonExample`.** Les triggers `executeWorkflowTrigger` en mode `jsonExample` droppent silencieusement tout champ absent de l'exemple — mettre à jour l'exemple à chaque évolution de contrat.
9. **HTTP Request pour Claude.** Le node Anthropic natif n8n génère une erreur 404.
---
## Zones rouges — Ne pas toucher sans précaution
| Workflow | ID n8n |
|---|---|
| `memory-search` | `ZhUeKCo8nMp35gk1` |
| `AIRouter` | `Cu8SozYmondKM8RB` |
| `Maestro` | `OlrQO21u178SjgBK` |
| `WIKI→PGVECTOR TRUNCATE` | `OUWDFF1HUhBr0Vy1` |
| `INTAKEROUTER (delete Inbox)` | `0c1moJ1mw2Ipcp4G` |
---
## Décision ouverte
| # | Sujet | Action requise |
|---|---|---|
| DA-001 | `AFTRSNComptable-Finance` (#22) et `AFTRSNQualite-Compliance` (#23) — câblés mais non publiés | Karter doit décider : publier pour cohérence OU documenter la raison de non-publication |
| DA-002 | `ZZZ_20260702_tmp_hermes_reach_test` (#31) | À supprimer par Karter |
---
## Livrables de cette session
| Fichier | Description |
|---|---|
| `aftersun_architecture_mvp.html` | Architecture complète interactive — version française |
| `aftersun_architecture_mvp_en.html` | Architecture complète interactive — version anglaise |
| `PASSATION_AFTRSN_AutomationArchitecture_20260702.md` | Ce document |
---
## Prochaine session recommandée
**Objectif :** Construire le flowchart BPMN détaillé de W-1 (AFTRSN-Event-Creator) node par node, le valider avec Karter, puis implémenter dans n8n.
**Pré-requis avant de commencer :**
- Credential Google OAuth créé dans n8n (Gmail AFTRSN + Drive + Calendar + Sheets)
- Connexion testée sur un workflow existant
**À fournir au prochain assistant :**
- Ce fichier de passation
- La Bible 360° Lumina OS (PDF projet)
- Le Brand Identity AFTERSUN (PDF projet)
- Les deux fichiers HTML d'architecture (référence visuelle)
---
*Généré le 02.07.2026 · Session Architecture MVP AFTERSUN · Lumina OS v1.0*
---
## ADDENDUM — 02.07.2026 · Ajouts session
### Notion — Système de gestion des tâches & gate de validation
**Contexte :** Notion est déjà connecté à n8n (credential `Notion account` existant). La DB actuelle `AI Automation — Knowledge` (`6fac8cb891e44c749e37a86419826905`) sert de vue humaine uniquement (synchronisée depuis pgvector via PUBLISHNOTION). Ce rôle reste inchangé.
**Nouveau rôle pour AFTRSN :** Notion devient le **système de gestion des tâches opérationnelles** et le **point de validation humaine** dans les workflows. C'est là que Karter voit ce qu'il doit faire, valide ou exécute, et où le système détecte que la prochaine étape peut se déclencher.
**Architecture retenue — Notion comme gate de workflow :**
```
n8n crée une tâche Notion
        ↓
Karter agit (valide / exécute / approuve)
        ↓
Notion : status → "Done" / "Approved" / "Confirmed"
        ↓
n8n poll le statut (scheduled ou webhook Notion)
        ↓
Déclenchement automatique de l'étape suivante
```
**Implémentation recommandée :**
Créer une nouvelle DB Notion dédiée AFTRSN (séparée de la KB Knowledge) avec le schéma suivant :
| Champ | Type | Usage |
|---|---|---|
| `title` | Title | Nom de la tâche |
| `status` | Select | `Todo` · `In Progress` · `Done` · `Approved` · `Blocked` |
| `event` | Relation | Lié à la DB Events |
| `workflow_id` | Text | ID du workflow n8n qui attend |
| `next_trigger` | Text | Nom du workflow à déclencher quand Done |
| `due_date` | Date | Deadline |
| `priority` | Select | `Critical` · `High` · `Normal` |
| `created_by` | Text | `n8n-auto` ou `Karter` |
**Pattern de polling dans n8n :**
- Scheduled toutes les 15–30 minutes : lire les tâches Notion avec `status = Done` non encore traitées
- Déclencher le workflow suivant via `executeWorkflow`
- Marquer la tâche comme `Processed` (champ supplémentaire) pour éviter double-déclenchement
**Exemples de gates de validation dans les workflows AFTRSN :**
| Workflow | Tâche créée dans Notion | Condition de déclenchement suivant |
|---|---|---|
| W-2 Venue-Coordinator | "Valider la réponse venue [NomVenue]" | status = Approved → envoyer contrat |
| W-3 Artist-Coordinator | "Confirmer booking [NomArtiste]" | status = Confirmed → générer contrat |
| W-4 Marketing-Planner | "Valider content calendar 08.08" | status = Approved → lancer scheduling posts |
| W-5 Email-Responder | "Revoir draft email [expéditeur]" | status = Approved → envoyer email |
| W-6 Budget-Tracker | "Valider budget final 08.08" | status = Approved → closing report |
**Credential n8n :** `Notion account` — déjà existant, pas de nouvelle connexion nécessaire.  
**À créer :** Nouvelle DB Notion "AFTRSN — Tasks & Operations" (distincte de la KB Knowledge).
---
### Wix — Site web, contacts & marketing AFTRSN
**Contexte :** Le site AFTERSUN est hébergé sur Wix (`aftersunpeople.com`). Wix est la plateforme centrale pour :
- Les pages d'événements (landing pages)
- La newsletter et les inscriptions
- La liste des contacts / participants enregistrés
**Wix est la source de vérité des contacts AFTRSN** (pas de CRM tiers type Mailchimp pour l'instant — contrainte budgétaire assumée).
**Ce que ça implique pour les workflows :**
| Usage | Source | Workflow concerné |
|---|---|---|
| Liste contacts newsletter | Wix CMS / Members | W-4 Marketing-Planner |
| Inscriptions event | Wix Forms / Events | W-4 · W-1 (comptage attendus) |
| Envoi newsletter | Wix Email Marketing | W-4 (draft → envoi via Wix) |
| Landing page event | Wix Pages | W-4 (contenu à mettre à jour manuellement ou via Wix API) |
**Intégration n8n ↔ Wix :**
Wix expose une API REST (Wix Headless / Wix Data API). Les endpoints utiles pour AFTRSN :
```
GET  /wix/api/v1/contacts          → liste des contacts
GET  /wix/api/v1/events/{id}/rsvp  → inscriptions à un event
POST /wix/api/v1/email-marketing   → déclencher une campagne email
GET  /wix/api/v1/forms/submissions → soumissions formulaires
```
**Credential à créer dans n8n :** HTTP Request avec API Key Wix (à récupérer dans le dashboard Wix → Settings → API Keys).
**Recommandation MVP :**
- Pour le 08.08 : utiliser Wix manuellement pour les envois newsletter (pas le temps de câbler l'API complète)
- W-4 Marketing-Planner génère le **contenu et le copy** → Karter copie-colle dans Wix Email Marketing
- Post-08.08 : câbler l'API Wix dans n8n pour automatiser les envois
**Contrainte identifiée :** Wix Email Marketing est limité en fonctionnalités de segmentation et d'automatisation avancée. À monitorer — si la liste de contacts dépasse 500 contacts actifs, migrer vers Mailchimp ou Brevo sera à envisager.
**MCP Wix disponible :** Le MCP Wix (`mcp.wix.com/mcp`) est connecté dans Claude. Il peut être utilisé pour lire/écrire dans le site Wix directement depuis Claude ou un workflow qui passe par Hermes. À explorer pour la gestion du contenu des pages event.
---
### Mise à jour de l'ordre d'implémentation
L'ordre reste le même. Les ajouts Notion et Wix s'intègrent ainsi :
**Semaine 1 :** Créer la DB Notion "AFTRSN — Tasks & Operations" avec le schéma défini ci-dessus.  
**Semaine 2 :** Intégrer les gates Notion dans W-2 et W-3 (validation venue + artistes).  
**Semaine 3 :** W-4 génère le contenu → Karter publie manuellement via Wix Email Marketing pour 08.08.  
**Post-08.08 :** Câbler l'API Wix dans n8n pour automatiser les envois newsletter.
---
### ADDENDUM 2 — Notion : Espace dédié AFTERSUN People (complet)
#### Philosophie
Notion n'est pas uniquement un système de tâches. C'est **le quartier général opérationnel d'AFTERSUN People** — l'endroit où tout ce qui concerne la marque, les événements, les artistes, les finances, le marketing et les opérations est documenté, suivi et validé. Un humain doit pouvoir ouvrir Notion et comprendre l'état complet de l'entreprise sans ouvrir un autre outil.
n8n alimente Notion automatiquement. Karter agit dans Notion. Notion informe n8n pour déclencher les étapes suivantes.
---
#### Architecture de l'espace Notion AFTERSUN People
```
AFTERSUN People (Workspace)
│
├── 🏠 Home — Dashboard opérationnel
│
├── 📅 EVENTS
│   ├── DB : Events (pivot central)
│   ├── DB : Venues
│   ├── DB : Artists
│   └── DB : Event Tasks (checklist par event)
│
├── 💰 FINANCE
│   ├── DB : Budgets (1 par event)
│   ├── DB : Invoices & Payments
│   └── Doc : Budget Templates
│
├── 📣 MARKETING
│   ├── DB : Campaigns
│   ├── DB : Content Assets
│   └── DB : Newsletter Archive
│
├── ⚙️ OPERATIONS
│   ├── DB : Tasks & Validations (gate n8n)
│   ├── DB : Contracts
│   └── Doc : Runbooks & Procédures
│
├── 🤝 RELATIONS
│   ├── DB : Artist Directory (tous artistes connus)
│   ├── DB : Venue Directory (tous lieux connus)
│   └── DB : Partners & Sponsors
│
├── 📚 KNOWLEDGE
│   ├── DB : Post-Event Reports
│   ├── DB : Lessons Learned
│   └── Doc : Brand Guidelines & Tone of Voice
│
└── 🤖 AI OPERATIONS
    ├── Doc : Architecture n8n (lien vers livrables)
    ├── DB : Workflow Registry (état des 7 workflows MVP)
    └── Doc : Invariants & Zones rouges
```
---
#### Détail des bases de données
**DB : Events** — pivot central, toutes les autres DB y sont liées
| Champ | Type | Notes |
|---|---|---|
| `name` | Title | Ex : "AFTERSUN 08.08.2026" |
| `date` | Date | Date de l'event |
| `status` | Select | `Idea` · `In Progress` · `Confirmed` · `Live` · `Closed` |
| `venue` | Relation | → DB Venues |
| `artists` | Relation | → DB Artists |
| `budget` | Relation | → DB Budgets |
| `drive_folder_url` | URL | Lien Drive auto-créé par W-1 |
| `calendar_url` | URL | Lien Calendar auto-créé par W-1 |
| `expected_attendance` | Number | Jauge capacité |
| `ticket_link` | URL | Lien Wix event |
| `campaign` | Relation | → DB Campaigns |
| `tasks` | Relation | → DB Event Tasks |
---
**DB : Venues** — annuaire des lieux
| Champ | Type | Notes |
|---|---|---|
| `name` | Title | |
| `status` | Select | `Prospect` · `Contacted` · `Negotiating` · `Confirmed` · `Partner` · `Rejected` |
| `capacity` | Number | |
| `location` | Text | |
| `contact_name` | Text | |
| `contact_email` | Email | |
| `contact_phone` | Phone | |
| `fee` | Number | |
| `contract_signed` | Checkbox | |
| `contract_url` | URL | Drive |
| `notes` | Text | |
| `events` | Relation | → DB Events |
---
**DB : Artists** — annuaire des artistes
| Champ | Type | Notes |
|---|---|---|
| `name` | Title | |
| `genre` | Multi-select | |
| `status` | Select | `Prospect` · `Contacted` · `Negotiating` · `Booked` · `Confirmed` · `Past` |
| `fee` | Number | |
| `contact_email` | Email | Agent ou artiste direct |
| `agent_name` | Text | |
| `rider_received` | Checkbox | |
| `rider_url` | URL | Drive |
| `contract_signed` | Checkbox | |
| `contract_url` | URL | Drive |
| `set_time` | Text | |
| `soundcloud_url` | URL | |
| `instagram_url` | URL | |
| `notes` | Text | |
| `events` | Relation | → DB Events |
---
**DB : Event Tasks** — checklist opérationnelle par event
| Champ | Type | Notes |
|---|---|---|
| `title` | Title | |
| `status` | Select | `Todo` · `In Progress` · `Done` · `Approved` · `Blocked` |
| `priority` | Select | `Critical` · `High` · `Normal` |
| `due_date` | Date | |
| `event` | Relation | → DB Events |
| `workflow_id` | Text | ID n8n qui attend ce statut |
| `next_trigger` | Text | Workflow à déclencher quand Done |
| `created_by` | Select | `n8n-auto` · `Karter` |
| `processed` | Checkbox | Coché par n8n après déclenchement |
---
**DB : Budgets**
| Champ | Type | Notes |
|---|---|---|
| `event` | Relation | → DB Events |
| `venue_cost` | Number | |
| `artist_fees_total` | Number | |
| `marketing_budget` | Number | |
| `ops_cost` | Number | |
| `misc` | Number | |
| `total_expenses` | Formula | Somme des coûts |
| `revenue_tickets` | Number | |
| `revenue_other` | Number | |
| `total_revenue` | Formula | |
| `margin_€` | Formula | Revenue − Expenses |
| `margin_%` | Formula | |
| `status` | Select | `Draft` · `Approved` · `Closed` |
| `sheets_url` | URL | Google Sheets détaillé |
| `closing_report_url` | URL | Drive |
---
**DB : Campaigns**
| Champ | Type | Notes |
|---|---|---|
| `name` | Title | Ex : "Campagne 08.08 — Announce" |
| `event` | Relation | → DB Events |
| `channel` | Multi-select | `Instagram` · `Newsletter` · `WhatsApp` · `Stories` |
| `status` | Select | `Draft` · `Approved` · `Scheduled` · `Published` |
| `publish_date` | Date | |
| `copy` | Text | Texte généré par W-4 |
| `asset_url` | URL | Canva |
| `wix_campaign_id` | Text | Référence Wix Email Marketing |
---
**DB : Contracts**
| Champ | Type | Notes |
|---|---|---|
| `title` | Title | Ex : "Contrat Venue — Le Rex 08.08" |
| `type` | Select | `Venue` · `Artist` · `Sponsor` · `Supplier` |
| `party` | Text | Nom de la contrepartie |
| `event` | Relation | → DB Events |
| `status` | Select | `Draft` · `Sent` · `Signed` · `Expired` |
| `signed_date` | Date | |
| `drive_url` | URL | |
| `amount` | Number | |
---
**DB : Post-Event Reports**
| Champ | Type | Notes |
|---|---|---|
| `event` | Relation | → DB Events |
| `attendance_actual` | Number | |
| `revenue_actual` | Number | |
| `margin_actual` | Number | |
| `what_worked` | Text | |
| `what_to_improve` | Text | |
| `artist_feedback` | Text | |
| `venue_feedback` | Text | |
| `marketing_performance` | Text | |
| `next_event_recommendations` | Text | |
| `ingested_to_memory` | Checkbox | Coché par W-7 après ingestion pgvector |
---
**DB : Workflow Registry** (AI Operations)
| Champ | Type | Notes |
|---|---|---|
| `workflow_name` | Title | Ex : "W-1 AFTRSN-Event-Creator" |
| `n8n_id` | Text | ID technique n8n |
| `status` | Select | `To Build` · `In Progress` · `Active` · `Paused` · `Deprecated` |
| `criticality` | Select | `Critical` · `Important` · `Utility` |
| `trigger` | Text | |
| `last_run` | Date | |
| `notes` | Text | |
---
#### Règles de fonctionnement Notion ↔ n8n
1. **n8n écrit, Karter valide.** n8n crée les entrées automatiquement (events, tasks, campaigns, budgets). Karter ne saisit pas manuellement ce qui peut être auto-généré.
2. **Notion n'est jamais édité à la main pour ce qui alimente pgvector.** La KB Knowledge (`AI Automation — Knowledge`) reste un dérivé de Git. Les nouvelles DB AFTRSN opérationnelles sont indépendantes.
3. **Le statut est le signal.** Tout workflow n8n qui attend une validation humaine poll le statut d'une tâche Notion. Changer un statut dans Notion = déclencher une action dans n8n.
4. **Un event Notion = un event Drive = un event Calendar.** Les trois sont créés ensemble par W-1 et restent liés via les URLs dans la fiche event Notion.
5. **Les DB Artist Directory et Venue Directory sont permanentes.** Elles existent indépendamment des events et s'enrichissent au fil du temps — c'est la mémoire relationnelle d'AFTERSUN, complémentaire à pgvector.
---
#### À créer en priorité (semaine 1)
1. Créer l'espace Notion "AFTERSUN People" (nouveau workspace ou page racine dédiée)
2. Créer les 3 DB fondamentales en premier : **Events** · **Venues** · **Artists**
3. Créer la DB **Event Tasks** avec le champ `workflow_id` et `next_trigger` — c'est le cœur du mécanisme de gate
4. Lier toutes les DB entre elles via les champs Relation
5. Créer la page **Home / Dashboard** avec des vues filtrées : events en cours · tâches critiques · artistes à confirmer · budget en attente d'approbation
