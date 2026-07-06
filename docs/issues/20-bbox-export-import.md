# Title

`.bbox` bundle export and import

## Summary

Implement the portable bundle format per DESIGN §20.2: `blackbox export
--format bbox` writing a zip (manifest.json, session.json, events.jsonl,
blobs/) after redaction, and `blackbox import <file.bbox>` restoring it with
collision handling and zip-safety guards — plus the shared `export` command
scaffolding (ref resolution, `--no-redact` confirmation gate, output-path
logic) that issues 21/22 plug their formats into.

## Context

`.bbox` is the machine-to-machine sharing/archival format (US-6) and the
future team-mode wire format (ADR-001). Byte-stable round-trips are the
correctness bar. This issue also owns `src/cli/commands/export.ts`'s common
skeleton because it is the first exporter to land.

## Scope

- deps: `adm-zip@^0.5`, `@types/adm-zip`
- `src/export/bbox.ts`, `src/export/common.ts` (shared export pipeline:
  load session+events+blobs → redact → format dispatch)
- `src/cli/commands/export.ts`, `src/cli/commands/import.ts` (replace
  placeholders; export registers formats via a `Record<format, Exporter>`
  map that 21/22 extend)
- integration tests

## Detailed Requirements

1. `src/export/common.ts`:
   - `loadExportBundle(db, blobStore, sessionId) → {session, events,
     blobs: Map<hash,text>}`: all events seq-ordered with payload JSON
     parsed; blobs = every hash in event_blobs for the session, full text.
   - `runExport(opts)`: resolve ref (issue 11), load bundle, apply
     redaction per flags (DESIGN §16.7: default on; `--no-redact` warns and
     requires `--yes` or TTY confirm), dispatch to the format exporter,
     print output path + size + redaction summary table (rule/count).
   - Default output filename `./<source>-<nativeId8>.<ext>`; `-o` may be a
     file path or existing directory (then default name inside it);
     refuse to overwrite an existing file unless `--force-overwrite`
     (new flag, applies to all formats; documented in DESIGN §16.7 — update
     that section in this PR, docs-first).
2. bbox writer (`src/export/bbox.ts`):
   - Zip layout exactly per DESIGN §20.2. `manifest.json` fields:
     formatVersion 1, tool "blackbox", toolVersion (package version),
     exportedAt (ISO, injected — the only nondeterministic field),
     sessions [{id, eventCount}], redaction report.
   - `events.jsonl`: one canonical-JSON line per event — stable key order
     (sorted) so round-trips are byte-comparable; `$blob` refs preserved
     (post-redaction hashes).
   - `blobs/<hash>` entries store redacted text; only hashes referenced by
     the exported events.
   - Estimated uncompressed size > 200 MiB → warning log before zipping
     (DESIGN §20.2).
3. Import (`src/cli/commands/import.ts` + reader in bbox.ts):
   - Validate: zip opens; manifest parses; `formatVersion === 1` else
     `BlackboxError` ("bundle format N unsupported"); every entry name must
     be relative, no `..` segments, no absolute paths, no symlink entries —
     violation → abort, nothing written (DESIGN §24).
   - Per session in manifest: existing session id → skip with message
     unless `--force` (then `deleteSession` first — reuse issue 11 logic
     WITHOUT the ignored-flag side effect: imported sessions have no
     source_files rows).
   - Insert inside one transaction: session row (set `imported_from` =
     bundle filename, `imported_at` now; PRESERVE original `source`),
     events (verbatim rows — seq/ts/type/text/payload), event_blobs from
     `$blob` refs found by walking payloads, blob files via BlobStore.put
     (hash must equal entry name; mismatch → abort transaction, corrupt
     bundle error).
   - FTS populates via triggers automatically. Print imported ids +
     event counts.
4. Round-trip invariant (the headline test): export session →
   import into a FRESH data dir → export again → the two `events.jsonl`
   byte-identical AND blob sets identical (manifest differs only in
   exportedAt; assert field-level equality otherwise).

## Acceptance Criteria

- [ ] Round-trip test passes for a fixture session containing: blobs,
      file_changes, Japanese text, and redacted content.
- [ ] Redaction-applied bbox contains ZERO strings matching built-in rules
      (scan every zip entry in test).
- [ ] `--no-redact --yes` bypasses; `--no-redact` on non-TTY without
      `--yes` → exit 2.
- [ ] Collision: second import skips (message, exit 0); `--force` replaces
      (event counts equal fresh import).
- [ ] Zip-slip: hand-built malicious zip with `../evil` entry → abort,
      no file created outside data dir, exit 1.
- [ ] Corrupt bundle (blob hash mismatch) → transaction rolled back, no
      partial session (count assertions).
- [ ] Overwrite refusal + `--force-overwrite`; `-o <dir>` naming.
- [ ] `blackbox export <ref>` with future formats absent → html default
      attempts and fails cleanly until issue 21 lands: register `html` as
      "not implemented yet" stub in the format map HERE so the default
      errors with exit 1 and a clear message (test), replaced in 21.
- [ ] Lint/typecheck/CI green; DESIGN §16.7 updated with
      `--force-overwrite`.

## Validation

Local: export a real session to bbox, import it under a temp
`BLACKBOX_HOME`, `blackbox show` both, compare stats; attach the two stat
outputs to the PR.

## Dependencies

- 11 (resolve/delete/load), 19 (redaction)

## Non-goals

- No html/md/json formats (21/22 — but the dispatch map and common pipeline
  land here). No multi-session bundles in ONE command invocation (manifest
  supports the array shape for v2; v1 exports exactly one session).
- No streaming zip (adm-zip in-memory accepted; unknown #6 measured in 23).

## Design References

- DESIGN §16.7–16.8, §20.2, §21 (report embedding), §24 (zip guards),
  §27 (format versions), §29 unknown #6
