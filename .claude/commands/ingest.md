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
1. Read `CLAUDE.md`, `index.md`, and the tail of `log.md` to load the conventions and see what
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
       { "path": "<exact raw/ path>", "category": "<the wiki category/theme you filed it under>",
         "processed_at": "<YYYY-MM-DD>" }
   This is the handoff to the n8n `LUMINA — Archive — Raw→Drive` workflow: it tells n8n which raw
   files are safe to move to Google Drive (into the matching category folder) and then remove
   from raw/. Only list files you genuinely compiled — leave anything still pending OUT of the
   manifest. Do NOT delete anything from `raw/` yourself; n8n moves + deletes, and only after it
   confirms the Drive upload succeeded.
9. Do NOT run git commit or push — I handle versioning via the Obsidian Git plugin.

Finish with a short report: which sources you processed, which wiki pages you created vs.
updated, which orphan pages you fixed during the backlink audit, which entries you added to
`_archive_queue.json`, and anything ambiguous you skipped and want me to clarify.
