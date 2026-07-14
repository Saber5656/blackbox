# Title

Markdown and JSON export formats

## Summary

Implement the two text export formats via the issue-20 pipeline:
`--format json` (full-fidelity normalized document, blobs inlined) and
`--format md` (human-readable narrative transcript for pasting into PRs,
docs, and post-mortems) per DESIGN §20.1 and §20.4.

## Context

JSON is the interchange/debugging format (and the fixture for external
tooling); Markdown is the lightweight sharing format when a full HTML viewer
is overkill. Both run through redaction like every export.

## Scope

- `src/export/json.ts`, `src/export/markdown.ts` (register in the issue-20
  format map)
- tests

## Detailed Requirements

1. JSON format (DESIGN §20.1):
   - Document: `{formatVersion: 1, exportedAt, redaction, session,
     events: BlackboxEvent[]}` — events seq-ordered, `$blob` refs REPLACED
     by full text (no cap; this is the fidelity format), payload parsed
     (objects, not strings), stable sorted key order for determinism.
   - Always includes `raw` events (fidelity; `--include-raw` is a no-op
     here — document in help text).
   - Output streaming: build with `JSON.stringify(doc, keys-sorted
     replacer, 2)`; memory accepted (single session).
2. Markdown format (DESIGN §20.4):
   - Header block:
     ```
     # <title>
     
     | field | value |
     |---|---|
     | session | <source>:<nativeId> |
     | project | <projectDir> |
     | started / ended | … / … |
     | duration | … |
     | events | N (U user / A assistant / T tool calls / F file changes) |
     | tokens | in X / out Y |
     | redaction | applied (N replacements) | off |
     ```
   - Then one section per event:
     `## [<seq>] <HH:MM:SS> <type>` + ` — <tool>`/` — <path>` suffix when
     present; sidechain events get `(sidechain)` suffix.
   - Bodies: user/assistant/summary/error → blockquote (`> ` per line);
     thinking → `<details><summary>thinking</summary>` … `</details>`
     (raw text inside, indented, blank-line separated — details tags render
     on GitHub); tool_call → fenced block ` ```json ` of input (or
     displayCommand as ` ```sh `); tool_result → fenced ` ``` ` capped at
     100 lines with `… (+N lines truncated)` tail; file_change → header
     line `**<kind>** \`<path>\` (+a −d)` then ` ```diff ` fence (cap 200
     lines, same tail); token_usage/session_meta → single italic line.
   - `raw` events omitted unless `--include-raw` (then ` ```json ` capped
     20 lines).
   - Fence safety: bodies containing triple backticks → use four-backtick
     fences (longest-run+1 rule; test with a diff containing ``` inside).
   - Blob refs: inline up to 100/200-line caps from full blob text; note
     `(from blob <hash8>)` appended to the fence info string.
3. Both: default filename extension `.json` / `.md` (issue-20 naming);
   redaction default-on; `-o -` writes to stdout (add to common pipeline:
   `-o -` bypasses overwrite checks and prints NOTHING else to stdout;
   summary goes to stderr — test).

## Acceptance Criteria

- [ ] JSON: fixture export parses; blob text inlined byte-exact
      (post-redaction); deterministic across runs (fixed exportedAt);
      round-trip sanity: event count and seq order preserved; sorted keys.
- [ ] Markdown snapshot test per event type (fixture session covering all
      11 types); fence-escape test (``` inside diff); 100/200-line caps
      with correct remainder counts; thinking in `<details>`; Japanese
      intact.
- [ ] `--include-raw` toggles raw sections in md; no-op in json (help text
      asserted).
- [ ] `-o -` streams document to stdout with logs on stderr only
      (execa test capturing both).
- [ ] Redaction applied in both (secret fixture scan).
- [ ] Lint/typecheck/CI green.

## Validation

Local: `blackbox export <real ref> --format md -o -` piped into a pager;
paste a section into a GitHub gist preview to confirm rendering (manual,
checklist in PR).

## Dependencies

- 20 (pipeline, format map), 19 (redaction), 11 (session load)

## Non-goals

- No HTML-in-markdown embedding, no per-event permalinks, no partial-range
  export (`--from/--to seq` is v2).

## Design References

- DESIGN §16.7, §20.1, §20.4, §21
