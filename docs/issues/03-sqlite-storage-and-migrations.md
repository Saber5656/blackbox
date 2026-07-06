# Title

SQLite storage: schema v1, pragmas, migration runner

## Summary

Add better-sqlite3, implement `src/db/database.ts` (open, pragmas, version
assertions, migration runner) and migration `0001_init.sql` containing the
complete DESIGN §9 schema: sessions, events, blobs, event_blobs, source_files,
events_fts (trigram) with sync triggers, and the meta table.

## Context

The schema is the contract between the importer (issue 09), query layers
(11/12), server (14), and exports (20–22). It ships complete in one migration
so later issues never alter tables. FTS5 trigram choice is ADR-004.

## Scope

- deps: `better-sqlite3@^11`, `@types/better-sqlite3`
- `src/db/database.ts`, `src/db/migrations/0001_init.sql`
- tests `src/db/database.test.ts`

## Detailed Requirements

1. `openDatabase(config: Config, opts?: {readonly?: boolean}) → Db` where
   `Db` wraps a better-sqlite3 `Database` plus helpers:
   - Apply pragmas on every open: `journal_mode=WAL`, `synchronous=NORMAL`,
     `foreign_keys=ON`, `busy_timeout=5000`.
   - Assert `SELECT sqlite_version()` ≥ `3.34.0`; else `BlackboxError`
     ("SQLite X.Y.Z lacks the trigram tokenizer; rebuild better-sqlite3").
     Version comparison must be numeric per segment, not string.
   - Assert FTS5 availability by preparing a scratch
     `CREATE VIRTUAL TABLE temp.__fts_probe USING fts5(x)` then dropping it;
     clear error if unavailable.
2. Migration runner:
   - Migrations = files matching `NNNN_name.sql` under `src/db/migrations/`,
     copied verbatim into `dist/db/migrations/` by the build (`build` script:
     add a copy step, e.g. `cp -R src/db/migrations dist/db/`). Runner reads
     from a path resolved relative to the compiled module
     (`new URL('./migrations/', import.meta.url)`).
   - `meta.schema_version` holds the last applied number as text. Fresh DB:
     runner creates `meta` first, then applies all migrations in ascending
     filename order, each inside ONE transaction, updating `schema_version`
     in the same transaction.
   - DB with `schema_version` > max local migration → `BlackboxError`
     ("database created by a newer blackbox") per DESIGN §27; readonly opens
     skip migrations but perform the same version check.
   - `meta.format_version = '1'` and `meta.created_at` (ISO) inserted by
     0001.
3. `0001_init.sql`: exactly the DDL in DESIGN §9 (tables, indexes, FTS table
   with `tokenize="trigram case_sensitive 0"`, insert/delete triggers, WAL
   pragma lines excluded — pragmas live in code, not the migration).
4. Helpers on `Db`: `transaction<T>(fn) → T` (better-sqlite3 `.transaction`),
   `close()`, and typed prepared-statement accessors used later
   (`db.raw` escape hatch is allowed for issues 09+).
5. Tests:
   - Fresh open creates schema; `schema_version == '0001'`;
     re-open is a no-op (no duplicate application).
   - Insert a session + event → row visible in `events_fts` via
     `SELECT rowid FROM events_fts WHERE events_fts MATCH 'hello'` (trigram:
     use a ≥3-char token). Delete event → FTS row gone (trigger).
   - `ON DELETE CASCADE`: deleting a session removes its events.
   - Newer-schema refusal path (manually set schema_version to '9999').
   - UNIQUE(session_id, seq) violation raises.

## Acceptance Criteria

- [ ] All tests above pass on macOS and Linux CI (native module builds).
- [ ] `dist/` contains the migrations after `npm run build`; a packaged
      global install (`npm install -g .`) can open a fresh DB (manual check).
- [ ] Trigram works: FTS query for a Japanese 3+ char string inserted in a
      test matches (e.g. text containing `検索対象` matched by `検索`... note:
      `検索` is 2 chars → use `検索対` 3-char term in the test).
- [ ] No other issue's tables/columns are missing versus DESIGN §9 (review
      checklist item; diff the SQL against the design table by table).
- [ ] Lint/typecheck/CI green.

## Validation

1. `sqlite3 ~/.blackbox/blackbox.db '.schema'` after a manual run shows the
   §9 objects (manual, local).
2. Corrupt test: truncate the DB file mid-test-suite is NOT required (out of
   scope) — but opening a zero-byte file must initialize it as fresh.

## Dependencies

- 01, 02

## Non-goals

- No DAO/query functions beyond the runner and helpers (issues 04/11/12).
- No migration 0002+ (schema is complete in 0001 for v1).
- No blob filesystem code (issue 04).

## Design References

- DESIGN §9 (schema), §27 (versioning)
- ADR-003 (better-sqlite3), ADR-004 (trigram)
