---
type: raw
title: "SOP_Configuration_OAuth2_n8n_Google"
source_url: "drive:1OSzve26W0_Jios-qSGK7vwjAsQOQViCh"
captured: 2026-06-22
vault: ai-automation
brand: null
immutable: true
---

# Standard Operating Procedure (SOP) : Configuration de l'Authentification OAuth2 Google dans n8n

## 1. Contexte et Objectif
Cette procédure documente la configuration pas à pas d'une connexion d'authentification sécurisée (OAuth2) entre une instance n8n auto-hébergée (`https://n8n.aftersunpeople.com`) et les API de l'écosystème Google Workspace (Google Drive, Gmail, etc.). L'objectif est de permettre à n8n d'accéder légitimement aux ressources Google pour exécuter des workflows automatisés sans compromettre la sécurité.

---

## 2. Processus de Configuration Étape par Étape

### Étape 1 : Création de l'application sur Google Cloud Console
1. Connectez-vous sur la [Google Cloud Console](https://console.cloud.google.com/).
2. Créez un nouveau projet dédié ou sélectionnez un projet existant.
3. Accédez à la section **Écran de consentement OAuth** (OAuth consent screen) :
   * Choisissez le type d'utilisateur **Externe** (External).
   * Remplissez les informations obligatoires (Nom de l'application, adresse email de support).
   * Laissez le projet en mode **Test** (Publishing status: Testing).

### Étape 2 : Génération des Identifiants (Credentials)
1. Dans le menu latéral gauche, cliquez sur **Identifiants** (Credentials).
2. Cliquez sur **+ Créer des identifiants** et sélectionnez **ID de client OAuth 2.0**.
3. Dans le menu déroulant *Type d'application*, sélectionnez impérativement **Application Web**.
4. Configurez la section **URI de redirection autorisés** (Authorized redirect URIs) :
   * Cliquez sur **+ Ajouter un URI**.
   * Collez l'URL exacte fournie par votre instance n8n : `https://n8n.aftersunpeople.com/rest/oauth2-credential/callback`.
   * *Note : Laissez la section "Origines JavaScript autorisées" vide.*
5. Cliquez sur **Créer**. Un pop-up s'affiche contenant votre **Client ID** et votre **Client Secret**. Copiez-les précieusement.

### Étape 3 : Configuration et Validation dans n8n
1. Ouvrez votre interface n8n et créez un nouveau nœud (ex: Google Drive ou Gmail).
2. Dans la section *Credential*, ajoutez une nouvelle configuration.
3. Collez l'**ID de client** (qui se termine par `.apps.googleusercontent.com`) dans la case **Client ID**.
4. Collez le **Code secret du client** dans la case **Client Secret**.
5. Cliquez sur **Sign in with Google**.
6. Sélectionnez le compte Google autorisé, ignorez l'avertissement de sécurité (cliquez sur *Paramètres avancés* -> *Accéder à n8n (non sécurisé)*) et validez l'accès.

---

## 3. Challenges Rencontrés et Résolutions

| Problème / Message d'erreur | Cause Racine | Solution Appliquée |
| :--- | :--- | :--- |
| **Error 401: invalid_client** <br>*(Client missing a project id)* | L'adresse email de l'utilisateur (`...@gmail.com`) a été collée par erreur dans le champ **Client ID** à la place de l'identifiant technique Google Cloud. | Remplacer l'email par le véritable jeton alphanumérique généré par Google Cloud finissant par `.apps.googleusercontent.com`. |
| **Error 403: access_denied** <br>*(Le compte n'est pas autorisé)* | L'application est en mode "Test" sur Google Cloud et l'adresse email tentant de s'authentifier n'a pas été déclarée comme testeur. | Aller sur l'écran de consentement OAuth sur Google Cloud, faire défiler jusqu'à **Utilisateurs de test**, cliquer sur **+ ADD USERS** et ajouter l'adresse email complète. |
| **Erreur de redirection de l'URI** | L'URL de callback entrée dans Google Cloud était incorrecte ou incomplète. | Configurer l'URI stricte au format : `https://<domaine>/rest/oauth2-credential/callback`. |

---

## 4. Astuces et Bonnes Pratiques

* **Le réflexe Navigation Privée :** Si le pop-up Google se connecte automatiquement au mauvais compte sans vous donner le choix, ouvrez votre instance n8n dans une **fenêtre de navigation privée**. Cela forcera Google à vous demander vos identifiants à partir d'une page vierge.
* **Gestion des environnements de Test :** Tant que l'application Google Cloud n'est pas publiée en production (ce qui requiert une vérification stricte par Google), les jetons d'accès (refresh tokens) expirent généralement tous les **7 jours** pour les utilisateurs de test externes. Pensez à ré-authentifier le nœud si vos flux s'arrêtent soudainement après une semaine.
* **Sécurité des clés :** Ne partagez jamais le *Client Secret* publiquement. Utilisez les variables d'environnement de n8n si vous déployez l'outil en équipe.

---

## 5. Leçons Apprises

1. **Différencier l'identité de l'identifiant :** Dans les plateformes d'intégration comme n8n, un "Client ID" n'est jamais une adresse email d'utilisateur, mais une clé d'identification machine-à-machine.
2. **Suivre la structure d'erreur HTTP :** * Une erreur **401** pointe toujours vers un problème d'identité de l'application (Mauvais Client ID ou Secret).
   * Une erreur **403** pointe vers un problème de droits d'accès ou de permissions (Utilisateur non autorisé, scope manquant).
3. **L'importance des URLs de redirection :** Google applique une vérification au caractère près sur l'URL de redirection. L'omission du segment `/rest/` dans les versions récentes de n8n est une source fréquente d'échec d'authentification.
