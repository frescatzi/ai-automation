---
type: raw
title: "AFTRSN-LUMINA — POS EXACT — Synchronisation mobile Obsidian CouchDB LiveSync — 2026-07-08"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn-lumina — pos exact — synchronisation mobile obsidian couchdb livesync — 2026-07-08.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

---
type: POS-EXACT
title: "AFTRSN-LUMINA — POS EXACT — Synchronisation mobile Obsidian (CouchDB self-hosted + plugin Self-hosted LiveSync)"
scope: AFTRSN-LUMINA
statut: actif
date: 2026-07-08
related:
  - "save-pos-md-to-drive"
  - "lumina-bible-360"
---

# POS EXACT — Synchronisation mobile Obsidian via CouchDB self-hosted + LiveSync

## 1. Objectif

Synchroniser les 3 vaults Obsidian de Lumina (`ai-automation`, `brands`, `personal` — clones Git de `frescatzi/ai-automation`, `brands`, `personal` sous `~/Documents/Lumina AI/GitHub/`) entre plusieurs appareils : Mac (desktop), **Obsidian ODB** (instance Obsidian conteneurisée sur Coolify, accessible par navigateur, utile comme nœud toujours disponible), et iPhone — sans dépendre d'Obsidian Sync (payant) ni de GitHub directement depuis le mobile.

**Solution retenue** : un serveur **CouchDB self-hosted** (sur Coolify/Hetzner) + le plugin Obsidian **« Self-hosted LiveSync »** (vrtmrz) installé sur chaque appareil. CouchDB ne remplace pas GitHub : GitHub reste la source de vérité pour la mémoire des agents (pgvector/Notion, déclenchés uniquement par un push Git — voir §7 « Points de vigilance »). CouchDB/LiveSync ne fait que garder les fichiers synchronisés en direct entre appareils ; le commit/push vers GitHub reste un geste séparé (fait depuis le Mac, ou à automatiser plus tard).

## 2. Infrastructure CouchDB (Coolify)

- **Projet Coolify** : AFTRSN-Automation → production.
- **Service** : conteneur `couchdb:latest`, déployé d'abord comme ressource « Docker Compose », **converti en type « Application »** (bouton **« Convert to Application »**) — nécessaire car Coolify classe l'image `couchdb` comme « database » par défaut, ce qui masque le champ Domaine/FQDN (seul un port TCP brut est proposé pour les services classés « database »).
- **Variables d'environnement** : `COUCHDB_USER`, `COUCHDB_PASSWORD` (valeurs dans Coolify → service Couchdb → Environment Variables — **ne pas les reporter dans ce document**, cf. §8 sécurité).
- **Volumes persistants** : bind mounts sur `/opt/couchdb/data` et `/opt/couchdb/etc/local.d`.
- **Domaine** : `https://couchdb.aftersunpeople.com` — dans Coolify, le champ Domain de la page General doit inclure le port interne (`https://couchdb.aftersunpeople.com:5984`) pour que le proxy route au bon port du conteneur, **mais l'URL utilisée côté client (curl, plugin Obsidian) ne doit PAS inclure ce port** — juste `https://couchdb.aftersunpeople.com` (port 443 standard, terminaison TLS gérée par le proxy Coolify/Traefik, certificat Let's Encrypt automatique dès qu'un domaine est assigné).

## 3. Script d'initialisation CouchDB (à lancer après CHAQUE reset des volumes)

```bash
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh \
  | hostname=https://couchdb.aftersunpeople.com username=<COUCHDB_USER> password='<COUCHDB_PASSWORD>' bash
```

Configure : CORS (origins `app://obsidian.md, capacitor://localhost, http://localhost`), `single_node=true`, `max_document_size=50000000`, `max_http_request_size=4294967296`, `require_valid_user=true`, et crée les bases système (`_users`, `_replicator`).

**Vérification** : `curl -u <COUCHDB_USER>:'<COUCHDB_PASSWORD>' https://couchdb.aftersunpeople.com/` doit renvoyer `{"couchdb":"Welcome",...}`.

## 4. Une base CouchDB par vault (invariant strict)

| Vault Obsidian | Dossier local (Mac) | Base CouchDB | Statut au 08-07 |
|---|---|---|---|
| ai-automation | `~/Documents/Lumina AI/GitHub/ai-automation` | `lumina-ai-automation` | ✅ actif (renommé depuis `luminacouchdb` par réplication native, cf. §6) |
| brands | `~/Documents/Lumina AI/GitHub/brands` | `luminabrands` | 🔧 base créée, vault à reconnecter (était par erreur sur `luminacouchdb`) |
| personal | `~/Documents/Lumina AI/GitHub/personal` | `luminapersonal` | 🔧 base créée, vault à connecter/vérifier |

⚠️ **Ne jamais faire pointer deux vaults différents sur la même base CouchDB** — un vault = une base, sans exception. Un incident a eu lieu le 08-07 (`brands` et `personal` pointaient sur la base de `ai-automation`) : heureusement sans pollution locale (`git status` propre sur les deux), corrigé par création de bases dédiées.

⚠️ Ne pas confondre les vaults Git (`~/Documents/Lumina AI/GitHub/...`) avec les coffres iCloud du même nom (`~/Library/Mobile Documents/com~apple~CloudDocs/Obsidian/{AI-Automation,Brands,Personal}`) — ce sont des copies séparées, sans lien Git, à ne pas utiliser pour cette synchro.

## 5. Renommer une base CouchDB (pas de fonction native « rename »)

```bash
curl -u <USER>:'<PASS>' -X PUT https://couchdb.aftersunpeople.com/<nouveau-nom>
curl -u <USER>:'<PASS>' -X POST https://couchdb.aftersunpeople.com/_replicate \
  -H "Content-Type: application/json" \
  -d '{"source":"<ancien-nom>","target":"<nouveau-nom>"}'
```
Puis reconnecter chaque appareil au nouveau nom (§6), et supprimer l'ancienne base une fois tous les appareils confirmés migrés :
```bash
curl -u <USER>:'<PASS>' -X DELETE https://couchdb.aftersunpeople.com/<ancien-nom>
```

## 6. Connecter un appareil (plugin Self-hosted LiveSync)

1. Installer le plugin (Réglages → Plugins communautaires → chercher « Self-hosted LiveSync » de vrtmrz).
2. Wizard :
   - **Premier appareil pour un vault donné** → « I am setting this up for the first time ».
   - **Appareil suivant** → « I am adding a device to an existing synchronisation setup ».
3. Choisir **« Enter the server information manually »** (la Setup URI/QR Code ne franchit pas le presse-papier isolé d'Obsidian ODB — cf. §7).
4. Renseigner : URI `https://couchdb.aftersunpeople.com` (sans port) · Username/Password (valeurs Coolify) · **Database name = celui du tableau §4, jamais partagé entre vaults**. Cocher **« Use Internal API »** (contourne un blocage CORS observé spécifiquement dans le contexte navigateur d'Obsidian ODB).
5. E2EE : laissé désactivé pour l'instant (à activer plus tard comme étape contrôlée séparée).
6. **« Detect and Fix CouchDB Issues »** doit afficher « All checks passed successfully! » avant de continuer.
7. À l'étape de récupération des données :
   - **« Compare time and take newer »** (jamais « Overwrite all with remote files » si le serveur est vide ou pas sûr).
   - **« Keep local files even if deleted on remote »** (jamais « Delete local files if deleted on remote » dans le même cas).

## 7. Pièges rencontrés (à ne pas refaire)

- **`COUCHDB_USER`/`COUCHDB_PASSWORD` ne s'appliquent qu'au tout premier démarrage** sur un volume de données vide. Changer ces variables dans Coolify et redéployer **ne change rien** tant que les volumes persistants (`Persistent Storages`) ne sont pas vidés puis le service redéployé sur un volume neuf.
- CouchDB **verrouille temporairement le compte** après plusieurs échecs d'authentification rapprochés (`"Account is temporarily locked"`) — attendre quelques minutes, ne pas re-tester en boucle.
- **Caractères spéciaux dans le mot de passe** (ex. `$`) : sans risque dans les champs de formulaire (Obsidian, Coolify), mais **dangereux tels quels dans un terminal** — bash interprète `$xxx` comme une variable. Toujours entourer de guillemets simples en ligne de commande.
- **Nom de base CouchDB** : minuscules uniquement, pas d'espace, pas de caractère spécial, ne peut pas commencer par `_`.
- **`Access forbidden` persistant malgré des identifiants confirmés bons via curl** → activer **« Use Internal API »** dans les réglages du plugin (contournement CORS spécifique au contexte Electron/navigateur d'Obsidian).
- **Presse-papier d'Obsidian ODB (conteneur/navigateur) isolé** de celui du Mac — « Copy the current settings to a Setup URI » ne se colle pas sur un autre appareil physique ; utiliser la saisie manuelle à la place. Le QR Code reste utilisable pour l'iPhone (scan caméra, pas de presse-papier impliqué).
- **Classification Coolify « database » vs « application »** : un service basé sur une image connue (ex. `couchdb`) est classé « database » par défaut et n'expose qu'un port TCP brut, pas de champ Domaine/HTTPS. Cliquer **« Convert to Application »** pour débloquer le champ Domain et l'HTTPS automatique.
- **Une base = un vault, jamais partagée** (cf. incident §4).
- **Vaults iCloud vs vaults Git** : bien vérifier (`find ~/Documents -name .git`) quel dossier est le vrai clone Git avant de le connecter à LiveSync — les copies iCloud du même nom ne sont pas les vaults sources de vérité.
- **Le pipeline pgvector/Notion (`ORCHESTRATOR-SYNC`) ne se déclenche que sur un push GitHub.** Les éditions faites uniquement via LiveSync (mobile, ODB) ne remontent pas automatiquement vers GitHub/pgvector/Notion tant que quelqu'un (ou un script) ne fait pas `git commit && push` depuis un appareil git-tracké. À automatiser plus tard côté serveur.
- **`wiki/` reste le domaine du LLM** (règle établie) — les éditions mobiles via LiveSync doivent rester dans `raw/` ou du contenu personnel, pas dans `wiki/`.

## 8. Sécurité

Le mot de passe CouchDB actuel a été manipulé en clair à plusieurs reprises pendant la session de mise en place (tests curl, diagnostics). **Recommandation : le faire tourner (rotation) dans Coolify une fois la synchro des 3 vaults confirmée stable sur tous les appareils**, puis relancer le script d'init (§3) avec la nouvelle valeur.

## 9. État d'avancement (08-07-2026)

- [x] CouchDB déployé, HTTPS valide, script d'init OK.
- [x] Vault `ai-automation` : Mac connecté sur `lumina-ai-automation`.
- [ ] Vault `ai-automation` : Obsidian ODB à repointer sur `lumina-ai-automation` (encore sur l'ancien nom `luminacouchdb`).
- [ ] Vault `ai-automation` : iPhone à connecter sur `lumina-ai-automation`.
- [ ] Vault `brands` : reconnecter Mac sur `luminabrands` (était par erreur sur la base partagée) ; connecter ODB + iPhone.
- [ ] Vault `personal` : vérifier/connecter Mac sur `luminapersonal` ; connecter ODB + iPhone.
- [ ] Supprimer l'ancienne base `luminacouchdb` une fois tous les appareils migrés.
- [ ] Rotation du mot de passe CouchDB (§8).
- [ ] Automatiser le commit/push GitHub depuis les appareils synchronisés via LiveSync (pour ne pas perdre les éditions mobiles côté pgvector/Notion).

## 10. Renvois

| Brique | Document |
|---|---|
| Version générique (sans détails AFTRSN) | `POS-GENERIQUE_configuration-obsidian-ios-self-hosted.md` |
| Architecture des 3 vaults | `wiki/architecture/brief-montage-vaults.md`, `Architecture_Connaissance_Obsidian_Centric.md` |
| Bible 360° (addendum infra) | `BIBLE_LUMINA-OS_360_2026-07-02.md`, ADDENDUM v1.3 |
