---
description: Compile new/updated raw/ sources into clean, fully back-linked wiki/ pages, then queue them for archiving
---

You are maintaining the AFTRSN knowledge vault (a Git repo opened as an Obsidian vault).
Your job: turn new or updated source material in `raw/` into clean, deduplicated, fully
cross-linked `wiki/` pages.

Rules:
- FIRST read `CLAUDE.md` at the repo root and follow its wiki conventions and workflows exactly.
  It is the schema and overrides these instructions if anything conflicts.
- `raw/` is the READ-ONLY source of truth — never edit, move, or delete anything inside it.
- You only write inside `wiki/`, `index.md`, `log.md`, and `_archive_queue.json`.

Steps:
1. PURGE DU MANIFESTE — avant toute autre action, en trois sous-étapes STRICTEMENT dans cet ordre :

   a) SYNC VÉRIFIÉE (garde-fou)
      Run: `git fetch origin`
      Then compare local HEAD to origin/main:
        `git rev-list --count HEAD..origin/main`  (commits remote has that local doesn't)
        `git rev-list --count origin/main..HEAD`  (commits local has that remote doesn't)
      If EITHER count is non-zero → local is behind, ahead, or diverged.
      → DO NOT PURGE. Print exactly:
          "Purge sautée : repo local non synchronisé avec origin/main — synchroniser d'abord."
        Then skip to step 2 without touching `_archive_queue.json`.
      Only proceed to (b) if both counts are 0 (perfectly aligned).

      Note for /sync runs: the `git pull` in sync step 1 must have completed as a fast-forward
      BEFORE this check is reached. If the pull was a no-op or failed, treat as not aligned.

   b) PURGE SUR DISQUE RÉEL (only if sync confirmed)
      Read `_archive_queue.json` (start from [] if missing).
      For EACH entry, physically test whether the file at `path` exists on disk in `raw/`:
        - ABSENT  → remove entry (already archived by n8n)
        - PRESENT → keep entry (pending or failed, must retry)
      Rewrite `_archive_queue.json` with the resulting array (write [] if all were removed).

   c) TRACE
      Print each entry with its status: "présent" or "absent".
      Then print: manifeste avant (N entrées) → après (M entrées).

   Règle d'or : ne JAMAIS purger sur la base du nombre d'entrées ou d'une hypothèse —
   toujours lire le système de fichiers APRÈS une sync confirmée.

2. Read `CLAUDE.md`, `index.md`, and the tail of `log.md` to load the conventions and see what
   has already been ingested.
2. Scan `raw/` and identify what is new or changed since the last log entry.
3. For each new/changed source: extract the durable facts and fold them into the right `wiki/`
   page(s). Update existing pages instead of duplicating; create a new page only when no good
   home exists. Follow the naming, structure, and [[wikilink]] conventions from CLAUDE.md.
4. Keep pages atomic and cross-linked; merge or de-duplicate overlapping content you find.
5. BACKLINKS — do not skip this:
   - Whenever a page mentions another concept, person, project, or page that has (or should have)
     its own wiki page, link it with [[wikilink]].
   - Make links reciprocal: if page A links to B, make sure B also points back to A where the
     relationship is meaningful (e.g. a "Related" / "See also" line).
   - Every concept page referenced via [[...]] must actually exist — if it doesn't, create at
     least a short stub page for it so there are no broken/red links.
   - After ingesting, do a backlink audit pass over the whole `wiki/`: find any orphan pages
     (no inbound links from any other page) and connect them by adding the appropriate links
     from related pages. No wiki page should be left without backlinks.
6. Update `index.md` so every wiki page is listed.
7. Append ONE dated entry to `log.md` summarizing what you ingested and which pages changed
   (append-only — do not rewrite past entries).
8. ARCHIVE MANIFEST — for EVERY `raw/` source you actually compiled in this run, append an entry
   to `_archive_queue.json` at the repo root (create the file if missing; keep it a valid JSON
   array). Each entry:
       { "path": "<exact raw/ path>", "category": "<see controlled vocabulary below>",
         "processed_at": "<YYYY-MM-DD>" }

   VOCABULAIRE CONTRÔLÉ — champ "category" (valeur EXACTE, accents/casse/parenthèses compris) :
     - IA & LLM
     - Agents & MCP
     - Automatisation (n8n)
     - Mémoire & Connaissance
     - Infrastructure
     - Architecture & Stratégie
     - Méthodes & SOP
   Règle : choisir la MEILLEURE correspondance unique parmi ces 7 valeurs. Si aucune ne colle
   parfaitement, prendre la plus proche. Ne jamais créer une nouvelle catégorie hors liste.

   This is the handoff to the n8n `LUMINA — Archive — Raw→Drive` workflow: it tells n8n which raw
   files are safe to move to Google Drive (into the matching category folder) and then remove
   from raw/. Only list files you genuinely compiled — leave anything still pending OUT of the
   manifest. Do NOT delete anything from `raw/` yourself; n8n moves + deletes, and only after it
   confirms the Drive upload succeeded.
9. Do NOT run git commit or push — I handle versioning via the Obsidian Git plugin.

Finish with a short report: which sources you processed, which wiki pages you created vs.
updated, which orphan pages you fixed during the backlink audit, which entries you added to
`_archive_queue.json`, and anything ambiguous you skipped and want me to clarify.
