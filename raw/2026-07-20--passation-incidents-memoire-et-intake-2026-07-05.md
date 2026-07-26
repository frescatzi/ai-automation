---
type: raw
title: "PASSATION_Incidents-memoire-et-intake_2026-07-05"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/passation_incidents-memoire-et-intake_2026-07-05.md"
captured: 2026-07-20
vault: brands
brand: AFTRSN
immutable: true
---

# PASSATION — Incidents mémoire & intake + plan de reprise (2026-07-05)

**Contexte :** en traitant la demande « pousser dans la Lumina Inbox tous les POS/marches à suivre non ingérés », une chaîne d'anomalies a été découverte, diagnostiquée et partiellement corrigée. Ce document permet à une nouvelle session de reprendre exactement où on s'est arrêté.

---

## 1. Incidents découverts et corrigés ✅

### 1.1 Credential Postgres corrompue (RÉSOLU — par Karter)
- `LUMINA_Postgres` (id `mlN3noHH3TT9C6Eu`) avait son blob chiffré à NULL (corruption entre 18h et 21h le 05-07). Tout Postgres était down : memory-search en 500, consolidation en erreur à 21:00, Chat Memory des agents KO.
- Symptôme trompeur : « Credential with ID … does not exist » à l'exécution alors qu'elle apparaît dans la liste ; `GET /rest/credentials/<id>` → 500 « Cannot destructure property 'data' ».
- **Fix : Karter a resaisi les valeurs dans la credential existante (sans en créer une nouvelle).** Vérifié : memory-search 200. Nota : l'ancienne « Postgres account » (`Ojp0aTYdIFfNsr4u`) n'existe plus.
- Cause racine non identifiée — si ça se reproduit, suspecter une édition de credential interrompue ou un souci de clé de chiffrement n8n.

### 1.2 Bombes TRUNCATE désamorcées (RÉSOLU)
- `LUMINA-MEMORY-INGESTION/WIKI-GITHUB→PGVECTOR` (`OUWDFF1HUhBr0Vy1`) faisait `TRUNCATE lumina_memory;` et `AFTRSN-MEMORY-INGESTION/DRIVE→AFTRSN_MEMORY` (`krWBy2Sj9k47H24z`) faisait `TRUNCATE aftrsn_memory;`.
- Héritage d'avant les collections : un simple push Git aurait **effacé canon, skills/leçons, insights, anomalies**.
- **Fix appliqué et republié :** `DELETE FROM <table> WHERE collection IN ('','docs') OR collection IS NULL;` — seuls les documents wiki sont régénérés, le reste survit.

### 1.3 Console SQL réparée (RÉSOLU)
- `LUMINA-UTIL-CONSOLE SQL` pointait vers une credential morte. Re-pointée vers `LUMINA_Postgres`. Requête en place : liste des `source_ref` distincts des deux banques — c'est l'outil pour répondre à « qu'est-ce qui est ingéré ? ».

## 2. Le vrai problème de la tâche initiale (DIAGNOSTIQUÉ, non corrigé)

1. **Aucun POS / marche à suivre n'est en mémoire.** Vérifié par SQL : 58 réfs = wiki de juin + canon + skills + épisodes. Candidats : **18 docs uniques à la racine du projet** (POS-AFTRSN_*, POS-GENERIQUE_*, POS-LUMINA_*, + Marches-a-suivre/POS-GENERIQUE_configuration-obsidian-ios) et **15 en archives** (archive-2026-06 : POS-EXACT, POS-GENERIQUE Runbook, Marches à suivre de juin).
2. **Le classifieur de l'intake rejette les POS.** `OPEN-AI-CALL` (gpt-4o-mini) a répondu `vault=reject, reason="Rien de pertinent"` au POS Bibliothèque-de-skills alors que le contenu (3 253 chars) lui était bien transmis. Raté déterministe, répété à chaque run.
3. **Les rejets bouclent :** un fichier rejeté RESTE dans l'Inbox et est re-téléchargé + re-classifié **chaque heure** (le POS ci-dessus depuis le 02-07). 9 fichiers en attente dans l'Inbox au 05-07 23h50, dont 2 doublons poussés le 05-07 par une autre session.
4. **Structurel : pousser dans l'Inbox n'ingère PAS.** La chaîne est Inbox → GitHub `raw/` (auto) → wiki (MANUEL : `/ingest` via Claude Code) → pgvector `docs` (auto au push). Sans l'étape manuelle, rien n'atteint la mémoire.

## 3. Plan de reprise (validé dans son esprit par Karter — point 1 prêt à lancer)

1. **Ingérer les 18 POS racine directement en mémoire**, `collection='procedures'` dans `lumina_memory` — SANS passer par l'Inbox. Méthode : les fichiers sont déjà sur le Drive (dossiers `POS-GENERIQUE` `1BQ4fArzenVY-VRslrZR8s5Wzq-X5J5hr`, `POS-EXACT` `1haw1aQ-FXjWyAl3ni8N2TbW7b2ZqYv0J`, + docs récents à la racine Drive) → exécuter `AFTRSN-MEMORY-INGESTION/DRIVE-RECURSIVE` (`8h3AowpjR9IBaxYF`) en copie de run paramétrée (folder id + table `lumina_memory` + collection `procedures` dans le workflowData posté), ou à défaut `LUMINA-MEMORY-INGEST-TEXT` fichier par fichier. ⚠️ Piège : `POST /rest/workflows/:id/run` → le pinData doit être DANS `workflowData.pinData` ; credentials résolues côté serveur depuis le workflow SAUVÉ.
2. **Archives (15 docs) : décision Karter en attente** — elles contiennent l'ancien roster (Brand-Guardian, etc.) : risque de contradiction avec le canon si ingérées telles quelles.
3. **Classifieur** : ajouter au prompt d'`OPEN-AI-CALL` (dans `LUMINA-INTAKE-ROUTER`, `0c1moJ1mw2Ipcp4G`) la règle : « POS / SOP / marche à suivre / runbook / passation → vault=ai-automation (ou brands si spécifique marque) ; ne jamais rejeter un document opérationnel Lumina/AFTRSN ».
4. **Stop à la boucle des rejets** : brancher la sortie `reject` de `If Rejected` sur un déplacement Drive vers un dossier `Lumina Inbox/Rejected` (+ 1 ligne dans `anomalies`).
5. **Créer `LUMINA-UTIL-AUDIT-INGESTION`** : workflow manuel qui sort le diff « fichiers Drive/projet ↔ source_ref en base » en une exécution. La demande d'aujourd'hui doit coûter 30 secondes, pas 6 minutes.
6. **Sentinel étage 2 (heartbeat)** : l'absence d'exécution planifiée n'est pas détectée aujourd'hui (seuls les échecs le sont). Ajouter un contrôle « dernier run < X heures » pour intake/consolidation/archive → alerte Telegram.
7. **Nettoyage Inbox (réservé Karter)** : supprimer les doublons du 05-07 (`POS-GENERIQUE_Calendrier-Contenu…` et `POS-GENERIQUE_Capture-Connaissance…` en double) et l'xlsx benchmark (rejet légitime : binaire).
8. **Après le point 1** : mettre à jour SYSTEM-CANON (§4 : nouvelle collection `procedures`) + réingérer, et consigner l'incident dans la Bible (v1.2).

## 4. Où tout se trouve

- Détail des incidents : mémoire persistante Cowork (`lumina-incidents-05-07`) + ce document.
- Canon agents : SYSTEM-CANON v1.1 (ingéré aftrsn+lumina). Référence humaine : BIBLE 360° v1.1 (+ addendum sur le Drive).
- Boucle mentor Hermès, Sentinel, rapport hebdo : actifs depuis le 03-07 (cf. addendum Bible v1.1).
- Alertes : bot Telegram **LuminaOsBot**, chat_id `776345147`, credential « LuminaOsBot - Telegram ».

*PASSATION — 2026-07-05, Claude (Cowork). Tout ce qui est marqué RÉSOLU a été vérifié live dans n8n.*
