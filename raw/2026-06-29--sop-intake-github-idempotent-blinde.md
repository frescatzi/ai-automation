---
type: raw
title: "SOP_intake-github-idempotent-blinde"
source_url: "drive:1CWeQHODWIp8oYWYxp4_nuADmXLOxOjgP"
captured: 2026-06-29
vault: ai-automation
brand: null
immutable: true
---

---
title: "SOP — Intake : Drive → GitHub (idempotent & blindé)"
type: sop
status: active
tags: [n8n, github, drive, intake, idempotence, openai]
version: 1.1
last_review: 2026-06-25
---

# SOP — Intake : Drive → GitHub (idempotent & blindé)

## Objectif
Faire entrer automatiquement toute nouvelle connaissance dans le système : prendre les fichiers déposés dans une boîte **Google Drive**, les classer, les écrire dans **GitHub** (la source de vérité), puis **nettoyer la boîte** — le tout sans bloquer et **sans jamais perdre un fichier**.

## À quoi ça sert / pourquoi c'est important
Ce workflow est la **porte d'entrée** de toute la connaissance : sans lui, le wiki ne se remplit pas et toute la mémoire en aval (agents + humains) reste vide. Le cœur du travail est fait par un **classifieur (OpenAI)** qui décide, pour chaque fichier, à quelle marque / quel repo / quel chemin il appartient — ou s'il faut l'ignorer. La partie délicate, et la raison de cette SOP, c'est de **ne jamais perdre un fichier** : on ne le supprime de Drive qu'**après avoir confirmé** qu'il est bien dans GitHub. Ce « blindage » évite deux catastrophes vécues : le **deadlock** (un fichier déjà présent qui fige tout le lot) et la **perte de données** (supprimer la source alors que l'écriture a échoué). Une fois ce workflow solide, la connaissance entre toute seule et la chaîne mémoire prend le relais. C'est donc le **point de départ automatisé** de tout le système.

## Quand utiliser cette procédure
Tout pipeline « drainer une boîte d'entrée → écrire ailleurs → nettoyer la source » où un même fichier peut être re-traité (Drive→GitHub, inbox→stockage…).

## Pré-requis
- n8n avec credentials **Google Drive**, **OpenAI**, **GitHub**.
- Une boîte Drive dédiée (« inbox »).
- Repos GitHub cibles (ex. `ai-automation`, `brands`).

## Le pattern (la fin du workflow)
```
… (classement) → Create a file → Get a file (vérif) → IF (sha présent ?)
                                                          ├─ true  → Delete (Drive)
                                                          └─ false → ✗ rien (reste dans Drive)
```
**Principe :** on tente la création ; **puis on vérifie** la présence dans GitHub ; on **ne supprime de Drive que si c'est confirmé**. Qu'il vienne d'être créé OU qu'il existait déjà, le Get le confirme → on nettoie. Sinon (vraie panne) → on garde pour retry.

---

## Workflow complet : `LUMINA — Intake — Router (Drive→GitHub)`
Rôle global : draine l'inbox Drive, classe chaque fichier, l'écrit dans le bon repo GitHub, puis supprime la source en toute sécurité.

**Chaîne et rôle de chaque node :**
1. **Schedule Trigger** — lance le workflow à intervalle régulier (ex. chaque heure) → l'inbox se draine seule, sans clic.
2. **Search files and folders** (Drive) — liste **tous** les fichiers de l'inbox (traite le backlog ET les nouveaux), pas juste un.
3. **Download file** (Drive) — télécharge chaque fichier listé.
4. **Extract from File** — extrait le **texte** du fichier (pour pouvoir le classer et l'écrire).
5. **OPEN-AI-CALL** — le **classifieur** : OpenAI (`gpt-4o-mini`, `response_format: json_object`) décide marque / repo / chemin, et accepter ou rejeter. *(On a quitté Gemini, instable en 503 sur le free tier.)*
6. **Code_Parse** (Code, *Run Once for Each Item*) — parse la réponse JSON d'OpenAI en champs exploitables : `repo`, `path`, `fileContent`, `fileId`, `reject`… Lit `choices[0].message.content`.
7. **If Rejected** — aiguille : **accepté** → on écrit ; **rejeté** → on laisse tomber (rien à ranger).
8. **Create a file** (GitHub) — écrit le fichier dans le repo. **Settings → On Error = Continue** (un fichier déjà présent renvoie une erreur « sha » qu'il ne faut pas laisser bloquer le lot).
9. **Get a file** (GitHub, **As Binary OFF**, On Error = Continue) — **vérifie** que le fichier est bien dans GitHub. Lit l'original via pairing (`{{ $('Code_Parse').item.json.repo }}` / `.path`) car après Create `$json` a changé de forme.
10. **IF** (`{{ $json.sha }}` *is not empty*) — **confirmé dans GitHub ?** true = oui, false = non.
11. **Delete a file** (Drive, sur la sortie **true**) — supprime de l'inbox **seulement si confirmé** (`File ID = {{ $('Code_Parse').item.json.fileId }}`). Sortie **false** non branchée → le fichier reste dans Drive pour un prochain run.

> **Re-publier** le workflow après modif, pour que la version planifiée/active utilise le correctif.

---

## Bonnes pratiques
- **Ordre = Create → Get → IF → Delete.** Create doit être **juste après** « If Rejected » (sinon il perd `fileContent`).
- **Repository Name dynamique** (`{{ $json.repo }}`) si les fichiers vont dans plusieurs repos (ai-automation, brands…).
- **Pairing `$('Code_Parse').item.json.*`** pour les champs d'origine après un node qui change la forme de l'item.
- **Delete uniquement après confirmation Get** (sortie true).
- Tester les **deux branches** : un fichier déjà présent (→ true → delete) et un fichier neuf (→ créé → true → delete).

## Erreurs fréquentes (vécues)
- **« Invalid request. 'sha' wasn't supplied »** → on essaie de *créer* un fichier qui *existe déjà*. Solution : On Error = Continue + vérif Get au lieu de bloquer.
- **Deadlock en boucle** : Create en `On Error = Stop` → un seul conflit fige tout le lot → l'inbox ne se vide jamais. Solution : Continue.
- **« first argument must be a string… undefined »** sur Create → `fileContent` vide, car le Get a été inséré AVANT Create. Solution : Create d'abord.
- **Tout part sur la branche False** alors que les fichiers existent → le Get est en **As Binary = ON** (pas de `sha` en JSON). Solution : Binary OFF.
- **« No path back to node »** en testant un node seul → normal en test isolé ; se résout quand le workflow tourne en entier.

## Points critiques
- Ne **jamais** supprimer de la source sans la **confirmation Get** (sinon perte si l'écriture GitHub a échoué).
- `On Error = Continue` sur **Create** ET **Get**.
- `As Binary = OFF` sur le Get (sinon pas de `sha`).

## Adaptation à une autre marque / contexte
Remplacer la source (Drive→autre), l'owner/repo, et le classifieur. Le pattern **Write → Verify → Delete-if-confirmed** reste identique pour tout « drainer une source vers un stockage ».

## Version
v1.1 — 2026-06-25 (workflow complet documenté node par node + description « à quoi ça sert / pourquoi »)
