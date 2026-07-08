# Title

Session query layer and CLI: list, show, delete

## Summary

Implement `src/query/sessions.ts` (list with filters, get with stats,
prefix resolution, delete with blob-orphan cleanup and re-import guard) and
the three CLI commands `blackbox list`, `blackbox show`, `blackbox delete`
per DESIGN §14 and §16.2–16.5.

## Context

First user-facing read surface (US-1 verification) and the shared
session-resolution helper every later command (replay, export, server) uses.
Delete semantics interact with sync (`ignored` flag) and the blob store
(orphan cleanup, issue 04's `deleteOrphans`).

## Scope

- `src/query/sessions.ts` + tests
- `src/cli/commands/{list,show,delete}.ts` (replace placeholders)
- CLI integration tests (execa over a tmp DB seeded via issue 09 importer on
  fixtures)

## Detailed Requirements

1. `listSessions(db, filter) → {total, items: SessionSummary[]}`
   - filter: `{source?, projectSubstring?, since?, until?, limit=50,
     offset=0, sort='started_at', order='desc'}`; since/until compare
     against `started_at`; projectSubstring: SQL `LIKE '%…%'` on
     project_dir with `\` escaping.
   - SessionSummary = all session columns camelCased.
2. `getSession(db, id) → SessionDetail | null` where detail = summary +
   stats (DESIGN §14): `countsByType` (SQL GROUP BY), `topTools`
   (tool_call GROUP BY tool_name LIMIT 10), `topFiles` (file_change GROUP BY
   file_path with Σ additions/deletions LIMIT 20), `durationMs`
   (ended-started, null-safe), `sourceFile` (path from source_files),
   `parseErrorCount` (error events with origin parser — via
   `json_extract(payload,'$.origin')='parser'`).
3. `resolveSessionRef(db, ref) → string` (full session id):
   match order (a) exact id, (b) exact native_id, (c) unique prefix of
   native_id, (d) unique prefix of the part after `:` in id. Zero matches →
   `BlackboxError` exit 3 `session not found: <ref>`; multiple →
   exit 4 listing up to 10 candidates as `<nativeId8>  <source>  <title>` on
   stderr.
4. `deleteSession(db, blobStore, id, {forget}) → {events, blobsDeleted}`:
   single transaction: capture `source_files.path` rows for the session;
   `DELETE FROM sessions WHERE id=?` (events + event_blobs cascade); then
   `blobStore.deleteOrphans`; then `UPDATE source_files SET ignored=1` (or
   DELETE row when `forget`) for the captured paths. Return counts.
5. `blackbox list` (DESIGN §16.2): flags `--source --project --since --until
   --limit --json`. Table columns exactly: `ID` (native_id first 8),
   `SOURCE`, `TITLE` (≤60, `…`), `PROJECT` (basename of project_dir),
   `STARTED` (local `YYYY-MM-DD HH:MM`), `DUR` (`1h23m` / `4m05s` / `12s`),
   `EVENTS`, `TOOLS`, `FILES`, `TOKENS` (`<in/1000>k/<out/1000>k`, `-` when
   null). Footer `N of M sessions`. `--json` → `{total, items}` verbatim.
6. `blackbox show <ref>` (§16.3): summary block (label-aligned key: value
   lines for every session field) + three stats tables (counts by type, top
   tools, top files with `+a/-d`). `--json` → SessionDetail. `--events`
   flag: after the summary, stream ALL events rendered with the shared
   renderer from issue 13 — since 13 lands later, in THIS issue `--events`
   prints `error: requires replay renderer (not yet implemented)` exit 1;
   issue 13 replaces it (documented cross-reference in both issues).
7. `blackbox delete <ref>` (§16.5): interactive confirm
   `Delete <source>:<nativeId8> "<title>" (<events> events)? [y/N]` on TTY;
   non-TTY without `--yes` → exit 2 usage error. `--forget` per req 4.
   Prints `deleted <events> events, <blobs> blobs`.
8. Time formatting helpers in `src/util/time.ts`: `formatLocal(iso)`,
   `formatDuration(ms)` — unit-tested (the `4m05s` zero-pad case).

## Acceptance Criteria

- [ ] Unit tests: every filter combination (source+project+since), sort
      order, limit/offset math, prefix resolution all four branches +
      ambiguity error listing, delete returns exact counts.
- [ ] Integration: seed fixtures via importer → `blackbox list --json` totals
      match; `list --source codex` filters; `show <8-char-prefix>` resolves;
      `show --json` stats hand-verified constants (topFiles additions from
      the fixture diff); `delete --yes` then `sync` → session does NOT come
      back (ignored flag respected — this asserts issue 09's skip);
      `delete --forget` then `sync` → session re-imported.
- [ ] Orphan blobs removed on delete; blob shared with another session (test
      constructs one) is preserved.
- [ ] Exit codes: 3 unknown ref, 4 ambiguous (both asserted via execa).
- [ ] Lint/typecheck/CI green.

## Validation

Local: `blackbox list`, `blackbox show <recent>`, `blackbox delete` a
disposable session, `blackbox sync`, confirm no resurrection; attach outputs
(redacted titles fine) to PR.

## Dependencies

- 09 (imported data, ignored-skip), 04 (deleteOrphans)

## Non-goals

- No search (12), no replay rendering (13), no HTTP (14).
- No bulk delete / delete-by-filter (v2).

## Design References

- DESIGN §8.3 (identity/prefix), §14 (stats), §16.2–16.5, §10 (orphans)
