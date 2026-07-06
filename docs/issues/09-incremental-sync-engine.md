# Title

Incremental sync engine and `blackbox sync` command

## Summary

Implement the ingestion pipeline end-to-end: source scanning
(`src/ingest/scanner.ts`), the incremental offset-resuming importer
(`src/ingest/importer.ts`) wiring adapters → FileChangeExtractor → normalize →
SQLite transactions, and the `blackbox sync` CLI per DESIGN §13.1–13.2 and
§16.1. After this issue, real Claude Code and Codex history is imported and
re-imported idempotently.

## Context

This is the heart of the recorder. Every property in DESIGN §13.2
(idempotence, partial-line safety, rewrite detection, crash safety, error
budget) must hold literally — later issues (watch, delete, e2e) build on
them.

## Scope

- deps: `fast-glob@^3`, `proper-lockfile@^4`
- `src/ingest/scanner.ts`, `src/ingest/importer.ts`,
  `src/cli/commands/sync.ts` (replaces placeholder)
- integration tests under `src/ingest/importer.test.ts` using fixture trees
  copied into tmpdirs

## Detailed Requirements

1. Scanner: `scanSources(config, {sourceFilter?, extraRoot?}) →
   {adapter, root, files: string[]}[]` using fast-glob with adapter
   `filePatterns`, absolute paths, sorted for determinism. Roots that do not
   exist → debug log, skip silently (fresh machines).
2. Importer `runSync(db, config, opts) → SyncSummary`:
   - Acquire `proper-lockfile` lock on `<dataDir>/.lock`
     (`retries: 0`); busy → `BlackboxError` exit 1:
     "another blackbox process is writing (sync/watch running?)".
   - Per file, follow DESIGN §13.1 pseudocode exactly:
     stat-skip (size+mtime equal) → cheap no-op; rewrite detection
     (`size < bytes_ingested` OR first-line hash mismatch when
     `bytes_ingested > 0`) → delete that file's session (events cascade; do
     NOT touch blobs here) and reset offsets, then re-import from 0;
     `--full` forces the rewrite path for every file.
   - Read with a manual chunked reader (64 KiB chunks) tracking BYTE offsets
     of complete `\n`-terminated lines only; buffer the partial tail without
     consuming it (offset math must count the exact UTF-8 bytes consumed
     including the newline). Do not use readline (byte offsets required).
   - Pipeline per line: `adapter.parseLine` → for each draft →
     `extractor.process(draft)` → normalize (`finalizeEvent`) → stage row.
     At EOF: `extractor.finish()` → same path. Drafts stage in memory per
     file; a file's staged batch commits in ONE transaction:
     upsert session (INSERT … ON CONFLICT(id) DO NOTHING, then UPDATE merged
     `sessionPatch` non-null fields), insert events with
     `seq = COALESCE(MAX(seq)+1, 0)` continuing across resumes, insert
     event_blobs, recompute counter columns via `UPDATE sessions SET
     event_count = (SELECT COUNT(*)…)` style statements, update
     `source_files` (bytes/lines/size/mtime/first_line_hash/session_id/
     last_synced_at/error_count).
   - Files larger than 32 MiB of NEW data: commit in intermediate
     transactions every 5 000 staged events (offset advances with each
     commit — crash safety preserved).
   - `SyncSummary`: per source {filesScanned, filesImported, filesSkipped,
     filesRewritten, eventsAdded, parseErrors, durationMs} + totals.
3. `blackbox sync` command (DESIGN §16.1): flags `--source <name>`,
   `--root <dir>` (extra root for that run), `--full`, `--json`, `--quiet`.
   Human output: one summary table (util/table) + per-file `--verbose` debug
   lines. `--json`: SyncSummary as single JSON doc on stdout.
4. Ordering guarantee: events insert in file order; `seq` dense per session
   (0..n-1) — assert in tests.
5. Failure containment: I/O error on one file → log error, `error_count++`,
   continue with next file, summary lists failed files, exit code stays 0
   unless EVERY file failed (then 1).

## Acceptance Criteria

- [ ] Integration test — fresh import: copy both fixture trees into a tmp
      structure mimicking real roots, run `runSync` → sessions/events counts
      match fixture expectations exactly (hand-computed constants in test);
      seq dense; FTS rows present; blobs created for the >64 KiB fixture
      payloads with event_blobs rows.
- [ ] Idempotence: second run → zero eventsAdded, filesSkipped == all.
- [ ] Append resume: append 2 lines to a fixture copy → only 2 lines' events
      added; session ended_at/counters updated; seq continues.
- [ ] Partial line: append a line WITHOUT trailing newline → zero events;
      offsets unchanged for that tail; then append `\n` → event appears.
- [ ] Rewrite detection: truncate a synced file to a shorter length → session
      re-imported from scratch (old events gone, new seq dense from 0).
- [ ] Malformed line ends up as an `error` event; error_count persisted;
      sync exit 0.
- [ ] Lock contention: second concurrent runSync (spawned process or manual
      lock) → clean BlackboxError, exit 1.
- [ ] Crash safety: kill the process between intermediate commits (simulate
      by making the importer throw after N events in a test hook) → re-run
      resumes with no duplicate or missing events (total equals the
      uninterrupted-run total).
- [ ] `blackbox sync --json` shape stable & documented in the test.
- [ ] Real-data smoke (local, not CI): `blackbox sync` over real roots
      completes; PR includes summary counts (no content).
- [ ] Lint/typecheck/CI green.

## Validation

1. `npm test -- ingest/importer`
2. Local: `blackbox sync && blackbox sync --json` (second run all-skips);
   `sqlite3` spot-checks: `SELECT COUNT(*) FROM events;`,
   dense-seq query `SELECT session_id FROM events GROUP BY session_id
   HAVING COUNT(*) != MAX(seq)+1;` returns zero rows.

## Dependencies

- 03 (db), 04 (normalize/blob), 06+07 (adapters), 08 (extractor)

## Non-goals

- No watch loop (issue 10). No `list/show` output (issue 11).
- No re-import of `ignored` source files (delete semantics; issue 11 sets
  the flag, this issue must respect it — implemented here as a skip).

## Design References

- DESIGN §4 (pipeline), §13.1–13.2, §16.1, §22 (error budget), §28 (risks)
