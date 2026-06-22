---
type: raw
title: "concept-configurer-oauth2-automation"
source_url: "drive:1FyVNByNAyQYwToXqjNdxZvg1zYno4YVa"
captured: 2026-06-22
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Configurer OAuth2 entre une plateforme d'automation et un service cloud"
status: draft
publish: none
vault: ai-automation
brand: null
sources: [raw/2026-06-21--sop-configuration-oauth2-n8n-google.md, raw/2026-06-21--sop-oauth2-generique.md]
related: ["concept-n8n-credentials.md"]
updated: 2026-06-22
---

# Configurer OAuth2 (plateforme d'automation ↔ service cloud)

Connecter un outil d'automation (n8n, etc.) à un service cloud (Google, etc.) en OAuth2 suit **toujours le même schéma** : on crée une « application » côté fournisseur, on génère un **Client ID + Client Secret**, et on les colle dans la plateforme qui ouvre alors une fenêtre d'autorisation.

## Les 3 étapes universelles

1. **Créer l'app côté fournisseur** (ex. Google Cloud Console) → écran de consentement OAuth en mode **Test/Externe** → ajouter son email en **utilisateur de test**.
2. **Générer les identifiants** → type **« Application Web »** → coller l'**URI de redirection** fournie par la plateforme (ex. `https://<domaine>/rest/oauth2-credential/callback`) → récupérer **Client ID** + **Client Secret**.
3. **Lier dans la plateforme** → coller Client ID + Secret → **Sign in** → autoriser le bon compte.

## Les 3 erreurs classiques (et leur cause)

| Erreur | Cause racine | Correctif |
|---|---|---|
| **401 / invalid_client** | On a collé un **email** dans le champ Client ID au lieu du vrai jeton machine (`…apps.googleusercontent.com`) | Mettre le vrai Client ID technique |
| **403 / access_denied** | App en mode « Test », compte **non déclaré** comme testeur | Ajouter l'email dans **Utilisateurs de test** |
| **Redirect URI mismatch** | URL de callback **incorrecte** (vérif au caractère près, HTTPS, segment `/rest/`) | Recopier l'URI exacte des deux côtés |

## Règles à retenir

- **401 = problème d'identité de l'app** (Client ID/Secret) · **403 = problème de droits** (utilisateur/scope).
- Vérification de l'URI de redirection **au caractère près** (l'oubli de `/rest/` est un piège fréquent).
- **Client ID ≠ email** : c'est une clé machine-à-machine.
- **Jetons en mode Test** : ils expirent souvent **tous les 7 jours** pour les testeurs externes → ré-authentifier si un flux s'arrête après une semaine. Pour la prod, publier l'app.
- **Astuce** : si le popup connecte le mauvais compte, ouvrir en **navigation privée**.
- **Sécurité** : ne jamais partager le Client Secret ; le stocker dans le gestionnaire chiffré de la plateforme.

## Exemple concret (Lumina)

Connexion **n8n (`n8n.aftersunpeople.com`) ↔ Google Drive** : c'est cette procédure qui a permis au robot d'ingestion d'accéder à l'inbox Drive. Voir aussi [[concept-n8n-credentials]].

---

> Sources : SOP OAuth2 (version n8n+Google spécifique + version générique réutilisable), capturées 2026-06-21.
