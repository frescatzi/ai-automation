---
description: Pull the latest from GitHub, then compile any waiting raw/ files into wiki/ pages and queue them for archiving
---

You are syncing the AFTRSN knowledge vault (a Git repo opened as an Obsidian vault) and bringing
it fully up to date.

Steps:
1. PULL: at the repo root, run `git pull` so the local vault matches GitHub. Report what changed
   (files added/updated). If there are local uncommitted changes that would block the pull, stop
   and tell me instead of forcing anything.
2. PURGE DU MANIFESTE: immediately after the pull, apply the full purge defined in `/ingest`
   step 1 (garde-fou sync → purge disque réel → trace). The pull above counts as the fast-forward
   prerequisite for the sync check — if it was a no-op or failed, the purge will self-abort.
3. INGEST: then run the project's `/ingest` workflow — compile everything still waiting in `raw/`
   into clean, deduplicated `wiki/` pages, with FULL backlinks (reciprocal links, no broken/red
   links, and a backlink audit so no page is left orphaned), update `index.md` and `log.md`, and
   append every compiled raw file to `_archive_queue.json` (path + category + processed_at) for
   the n8n archiver. The "category" field MUST be one of the 7 controlled values defined in
   `/ingest` step 8 — verbatim, never a new value. `raw/` stays read-only; only write inside `wiki/`, `index.md`, `log.md`, and
   `_archive_queue.json`. Do NOT delete anything from `raw/` — n8n handles the move to Drive +
   deletion.
3. Do NOT git commit or push — I handle versioning via the Obsidian Git plugin.

Finish with a short report: what the pull brought in, which wiki pages you created vs. updated,
which orphan pages you fixed, which entries you queued in `_archive_queue.json`, and anything
ambiguous you skipped and want me to clarify.
