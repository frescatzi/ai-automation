---
type: raw
title: "SOP_GENERIQUE_workflow-archivage-idempotent-n8n"
source_url: "drive:1fMa6nhXmchjq66ry1jyFSXZDx74EoXA1"
captured: 2026-06-29
vault: ai-automation
brand: null
immutable: true
---

# POS générique — Workflow d'archivage idempotent (n8n)

> **Procédure Opérationnelle Standard réutilisable** pour tout workflow n8n qui, après compilation amont, **archive une source vers un stockage puis la supprime — sans perte ni doublon**. À adapter (repo, stockage, IDs) au cas concret.
> **Principe** : *l'amont étiquette (manifeste), n8n exécute.*

---

## 1. Objet

Déplacer des fichiers source déjà traités (listés dans un manifeste) vers un stockage d'archive, puis les retirer de la source **uniquement après confirmation de l'archivage**. La sortie validée reste la source de vérité ; la source brute devient transitoire ; l'archive est le filet de sécurité.

## 2. Déclenchement

- **Cron** (recommandé) avec **minute explicite** pour désynchroniser des autres workflows.
- Manuel pour les tests.

## 3. Pré-conditions

- Manifeste à jour et accessible (liste **uniquement** ce qui est prêt).
- Dossier d'archive **hors** de tout dossier surveillé par un workflow d'ingestion.
- Credentials source + stockage valides dans n8n.

## 4. Déroulé type

1. **Trigger** (cron).
2. **Lire le manifeste** (texte ; décoder si base64).
3. **Parser** → 1 item par fichier.
4. **Récupérer le fichier source** (binaire), `On Error = error output` → **Success = à archiver / Error = skip** (idempotence).
5. **Uploader** vers l'archive (`On Error = error output`).
6. **Supprimer de la source** — branché **uniquement sur Success de l'upload**.

## 5. Garde-fous obligatoires

1. Suppression **après confirmation** (branchée sur Success de l'upload).
2. **Seuls** les fichiers du manifeste sont touchés.
3. Archive **hors** du dossier surveillé (anti-boucle).
4. La suppression source **ne re-déclenche pas** les workflows aval (vérifier leurs filtres).

## 6. Vérification d'un run

- **Nominal** : fetch Success = N, Error = 0 ; upload = N ; suppression = N ; fichiers dans l'archive ; source vidée.
- **À vide** : fetch Success = 0, Error = N ; upload = 0 ; suppression = 0. **Attendu.**

## 7. Incidents & réponses

| Symptôme | Cause probable | Action |
|---|---|---|
| Lecture manifeste = 404 | Manifeste absent/non poussé | Régénérer + publier en amont. |
| Tout en Error au fetch | Sources absentes | Normal si déjà archivées ; sinon vérifier la publication des sources. |
| « no binary 'data' » à l'upload | Node intercalé qui perd le binaire | Upload **directement** après le fetch binaire ; nom via `$binary.data.fileName`. |
| Champs d'origine perdus | `$json` remplacé par binaire/upload | **Pairing** `{{ $('<Node>').item.json.x }}`. |
| Suppression sans archivage | Garde-fou contourné | Suppression doit partir de la sortie **Success** de l'upload. |

## 8. Récupération

Toujours s'assurer d'une **double récupérabilité** avant d'autoriser la suppression : copie dans l'archive **et** historique de version (Git ou équivalent).

## 9. Bonnes pratiques

- Construire et tester **node par node**, puis l'upload seul, puis brancher la suppression.
- Valider l'**idempotence** par un re-run complet avant d'activer.
- Garder les secrets **dans n8n** uniquement.
- Démarrer **simple** (archive à plat), raffiner ensuite (sous-dossiers, nettoyage du manifeste, trigger réactif).
