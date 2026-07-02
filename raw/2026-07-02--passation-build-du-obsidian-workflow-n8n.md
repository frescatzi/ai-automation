---
type: raw
title: "Passation Build du Obsidian-workflow n8n"
source_url: "drive:1ci1f5BxfwL4Jzta4JrlAwc3n388bMdv6"
captured: 2026-07-02
vault: ai-automation
brand: null
immutable: true
---

# Passation Build du Obsidian-workflow n8n — `LUMINA — Archive — Raw→Drive`
*Document de passation. À ouvrir dans une NOUVELLE conversation dédiée au build complet du workflow.
Objectif de la prochaine session : trancher les questions ouvertes et construire le workflow de bout
en bout dans n8n.*

---

## 1. Ce qu'on construit (en une phrase)
Après que Claude Code a compilé `raw → wiki`, ce workflow n8n archive chaque fichier brut original
dans **Google Drive, rangé par catégorie**, puis le **supprime de `raw/`** — pour que `raw/` reste
toujours vide et que `wiki/` soit la seule source qui fait foi. Le brut est conservé dans Drive comme
filet de sécurité (provenance).

## 2. Où ça s'insère dans le système
Pipeline Lumina : Drive Inbox → (intake n8n) → GitHub `raw/` → **Claude Code `/ingest` compile →
`wiki/` + manifeste `_archive_queue.json`** → push → cross-trigger pgvector + Notion.
**Ce nouveau workflow est le maillon manquant** : il consomme `_archive_queue.json` pour vider `raw/`
vers Drive. Principe directeur : **l'IA étiquette (Claude Code écrit la catégorie), n8n exécute.**

## 3. Déjà décidé (ne pas re-débattre)
- Archivage fait par **n8n** (pas Claude Code local).
- Rangement **par catégorie** dans Drive.
- `raw/` devient **transitoire** (vidé après archivage) ; `wiki/` = source canonique.
- Le brut n'est **pas supprimé**, il est **conservé dans Drive**.
- Garde-fou : **supprimer de `raw/` seulement après confirmation de l'upload Drive** (même pattern
  que l'intake).
- Garde-fou : le dossier d'archive Drive **doit être HORS de l'Inbox** surveillé par l'intake
  (sinon boucle infinie).
- n8n n'archive **que** les fichiers listés dans le manifeste (les fichiers `raw/` pas encore
  compilés sont laissés intacts).

## 4. Manifeste produit par Claude Code (`/ingest` et `/sync`)
Fichier `_archive_queue.json` à la racine du repo, ex. :
```json
[
  { "path": "raw/note-x.md", "category": "ai-automation", "processed_at": "2026-06-25" }
]
```
La valeur `category` = nom du sous-dossier Drive cible.

## 5. Questions ouvertes + recommandations (à confirmer en début de session)
1. **Déclencheur** — *Recommandé pour le MVP : Schedule Trigger (ex. toutes les heures).* Simple,
   robuste, évite les boucles de re-déclenchement liées aux pushs. (Alternative plus réactive :
   GitHub push filtré sur `_archive_queue.json` — à garder pour plus tard.)
2. **Nettoyage du manifeste** — *Recommandé : ne PAS nettoyer au début.* On se repose sur
   l'idempotence : si le fichier n'existe plus dans `raw/`, on skip. Les entrées périmées sont
   inoffensives. (Nettoyer le JSON via un write-back GitHub = optionnel, plus tard.)
3. **Arborescence Drive** — *Recommandé : un seul niveau* `Archive-Raw/<category>/`, catégories en
   miroir du classement wiki (ex. `ai-automation`, `brands`, `personal`). Le workflow cherche le
   sous-dossier par nom et le crée s'il manque.

## 6. Pré-requis à faire AVANT le build
- [ ] Créer dans Google Drive un dossier racine d'archive, ex. **`Archive-Raw/`**, **hors du dossier
      "Lumina Inbox"**. Noter son **folder ID** (ce sera un paramètre du workflow).
- [ ] Vérifier qu'au moins un fichier figure dans `_archive_queue.json` (lancer `/ingest` une fois)
      pour tester le workflow sur des données réelles.

## 7. Chaîne de nodes à construire (détail complet dans `Spec Obsidian-Workflow n8n.md`)
1. Trigger (Schedule recommandé).
2. Get a file (GitHub) → lire `_archive_queue.json`.
3. Code → `JSON.parse` → 1 item par fichier.
4. Filtre "encore présent dans `raw/`" (Get file ; 404 → skip) = idempotence.
5. Get a file (GitHub) → contenu du fichier `raw/`.
6. Google Drive → trouver/créer `Archive-Raw/<category>`.
7. Google Drive → upload du fichier.
8. IF upload confirmé → [true] Delete a file (GitHub) sur le chemin `raw/...` / [false] ne rien
   supprimer.

## 8. Infos techniques utiles (sans secrets)
- n8n self-hosté : `n8n.aftersunpeople.com` (Hetzner/Coolify).
- Repo GitHub : `frescatzi/ai-automation`, branche `main`. Cred GitHub **prédéfinie** dans n8n.
- Google Drive : cred déjà utilisée par l'intake. Drive **Lumina Inbox** (à NE PAS utiliser comme
  archive) : `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`.
- Nommage : workflow infra → **`LUMINA — Archive — Raw→Drive`** ; dossier n8n = infra LUMINA.
- ⚠️ Sécurité : ne jamais coller de token/PAT dans la conversation — les creds vivent dans n8n.

## 9. Prompt d'ouverture pour la nouvelle conversation
> On construit ensemble, étape par étape dans n8n, le workflow `LUMINA — Archive — Raw→Drive`.
> Lis ce document de passation et la spec `Spec Obsidian-Workflow n8n.md`. Commence par me faire
> confirmer les 3 questions ouvertes (déclencheur, nettoyage manifeste, arborescence Drive), puis
> guide-moi node par node. Je suis débutant sur n8n : explications concrètes, un node à la fois.
