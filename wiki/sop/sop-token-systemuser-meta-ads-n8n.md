---
type: wiki
title: "SOP — Token System User Meta Ads pour n8n (création de campagnes)"
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/2026-07-21--pos-generique-meta-ads-systemuser-token-n8n-20260705.md
related:
  - wiki/concept-n8n-credentials.md
  - wiki/concept-oauth2-automation.md
  - wiki/sop/Guide-Connexion-Agents-AI-n8n.md
updated: 2026-07-21
---

# SOP — Token System User Meta Ads pour n8n (création de campagnes)

> Générique — toute marque pilotée via LUMINA OS qui veut créer des campagnes Meta Ads (PAUSED) depuis n8n.

## Idée centrale

Meta Ads n'utilise **pas** le patron OAuth2 classique (voir [[concept-oauth2-automation]]) pour un accès machine-à-machine : on génère un **access token System User** longue durée (expiration « Jamais »), stocké dans un credential n8n natif **« Facebook Graph API »** (pas « (App) »). Le secret n'est jamais saisi par l'agent — l'humain crée le credential lui-même (même principe de séparation que documenté dans [[concept-n8n-credentials]]).

## Procédure (ordre exact)

1. **App Meta** (developers.facebook.com) : créer une app, cas d'usage **« Créer et gérer les publicités avec l'API Marketing »**, la rattacher au **portefeuille business** qui détient le compte publicitaire. Vérifier que `ads_management` apparaît (« Prête pour le test » suffit à ce stade).
2. **Publier l'app (mode Live)** : Tableau de bord → Publier. Prérequis : **URL de politique de confidentialité** + **catégorie** renseignées dans Paramètres de base. Aucune vérification d'entreprise n'est requise pour gérer ses propres comptes.
3. **Utilisateur système** : Business Settings → Utilisateurs système → Ajouter (rôle Employee suffit). Lui affecter le **compte publicitaire** (permission « Gérer les campagnes ») + la **Page**.
4. **⭐ Étape qui débloque tout** : Business Settings → **Applications → [l'app] → Affecter des personnes** → sélectionner l'utilisateur système → activer **« Gérer l'app » (Contrôle total)** → Enregistrer. Ni la publication de l'app ni l'affectation du compte pub ne suffisent sans ce rôle applicatif explicite.
5. **Générer le token** : Utilisateurs système → [user] → Générer un token → choisir l'app → expiration **Jamais** → les scopes `ads_management` (+ `business_management`) n'apparaissent qu'**après** l'étape 4 → générer. Le token n'est affiché qu'une seule fois.
6. **Credential n8n** : Credentials → New → **« Facebook Graph API »** (pas « (App) », qui attend un App ID/Secret) → coller l'access token seul.
7. **Créer une campagne** : `POST https://graph.facebook.com/v21.0/act_<ID>/campaigns`, credential `facebookGraphApi`, corps JSON minimal :
   `{"name":"…","objective":"OUTCOME_TRAFFIC","status":"PAUSED","special_ad_categories":[],"is_adset_budget_sharing_enabled":false}`

## Difficultés rencontrées → causes → correctifs

| Symptôme | Cause racine | Correctif |
|---|---|---|
| « Aucune autorisation disponible — affectez un rôle d'application à l'utilisateur système » (même app publiée + compte pub affecté) | Le rôle applicatif (étape 4) est distinct de l'affectation du compte pub | Affecter explicitement « Gérer l'app » à l'utilisateur système via Applications → Affecter des personnes |
| Bouton **Publier** grisé | Privacy policy URL et/ou catégorie manquantes | Renseigner les deux dans Paramètres de base avant de publier |
| Création de campagne → erreur générique **100 « Invalid parameter »** | Message Meta tronqué par défaut | Activer `fullResponse` + `neverError` sur le node HTTP pour lire `error_user_title` → révèle le champ manquant (`is_adset_budget_sharing_enabled`) |
| Confusion credential n8n | « Facebook Graph API (App) » attend App ID/Secret, pas un token | Choisir « Facebook Graph API » (token seul) |

## Leçons apprises

- Un token System User n'expose ses permissions (`ads_management`…) que si l'utilisateur système possède un **rôle dans l'app** (« Gérer l'app »). C'est le point de blocage n°1, non évident dans l'UI Meta.
- Publier l'app exige privacy policy URL + catégorie, mais **pas** de vérification d'entreprise pour gérer ses propres comptes.
- Pour toute erreur Meta 100 générique : capturer la réponse brute (`fullResponse` + `neverError`) plutôt que de deviner — `error_user_title` donne le champ exact manquant.
- **Créer les campagnes en `PAUSED`** ; adsets/creatives/budget restent manuels (Ads Manager) — la coquille automatisée suffit à cadrer la revue humaine avant dépense réelle.
- Secret jamais saisi par l'agent : l'utilisateur crée le credential n8n lui-même (passkey/2FA = humain uniquement), même principe que les autres credentials sensibles — voir [[concept-n8n-credentials]].
- Mettre le node Meta en **onError = continue** dans un workflow de contenu : un échec pub ne doit jamais bloquer la génération de contenu associée.

## Voir aussi

- [[concept-n8n-credentials]] — gestion des credentials n8n (API Key, OAuth2, séparation agent/humain sur les secrets).
- [[concept-oauth2-automation]] — patron OAuth2 générique ; le token System User Meta s'en distingue (pas de flux Authorization Code, expiration « Jamais »).
- [[sop/Guide-Connexion-Agents-AI-n8n]] — création d'autres credentials n8n (Anthropic/OpenAI/Gemini) pas à pas.
