# Title

Search engine (FTS5 query layer) and `blackbox search` command

## Summary

Implement `src/query/search.ts` — FTS5 trigram query building, filters,
snippet extraction, bm25 ranking, and the short-query LIKE fallback — plus
the `blackbox search` CLI per DESIGN §15 and §16.4. This delivers US-2
("which session touched X, and why").

## Context

The FTS table and triggers exist since issue 03; the importer populates them.
ADR-004 fixed the trigram tokenizer and the <3-char fallback. DESIGN §29
unknown #5 (snippet() on CJK) is resolved inside this issue.

## Scope

- `src/query/search.ts` + tests
- `src/cli/commands/search.ts` (replaces placeholder)
- ANSI highlight rendering in the CLI table

## Detailed Requirements

1. Query building `buildFtsQuery(userQuery) → {kind:'fts'|'like'|'empty',
   value}`:
   - Trim; empty → `empty` (CLI: usage error exit 2).
   - Split on Unicode whitespace. If ANY term has < 3 code points → `like`
     kind with the original terms (trigram cannot match it; DESIGN §15.1).
   - Else `fts`: each term double-quoted with internal `"` doubled, joined
     with ` AND `.
2. `search(db, params) → {total, hits}` with params
   `{query, types?, tools?, sources?, sessionId?, projectSubstring?,
   pathSubstring?, since?, until?, limit=50 (cap 200), offset=0}`:
   - FTS path: exactly the SQL shape of DESIGN §15.2 (snippet col 0,
     markers `char(1)`/`char(2)`, ellipsis `…`, 24 tokens; ORDER BY bm25,
     ties ts DESC). Filters compile to AND'd conditions with bound params
     only (no string interpolation).
   - LIKE path: same SELECT/filters but
     `e.text LIKE '%'||?||'%' ESCAPE '\'` per term (AND), rank = 0, snippet
     = hand-rolled: find first case-insensitive match offset in `e.text`,
     take 60 chars before/after on char boundaries, wrap match with the same
     char(1)/(2) markers, prefix/suffix `…` when cut.
   - `total` via COUNT(*) with identical WHERE.
   - Hits: `{event: {sessionId, seq, ts, type, toolName, filePath,
     textTruncated}, sessionNativeId, sessionTitle, sessionSource, snippet,
     rank}`.
   - **CJK snippet verification (unknown #5)**: a test inserts Japanese text
     and asserts `snippet()` returns the marked term; if SQLite's snippet is
     broken for trigram+CJK (garbled or empty), switch the FTS path to the
     hand-rolled snippet over `e.text` (decision recorded by updating DESIGN
     §15.2 in the same PR — docs-first rule applies to the outcome either
     way).
3. `blackbox search` (DESIGN §16.4): args = query words (joined with
   spaces); flags `--type --tool --source --session --project --path
   --since --until --limit --offset --json`.
   - Table columns: `SESSION` (nativeId8), `SEQ`, `TS` (local HH:MM),
     `TYPE`, `TOOL`, `SNIPPET` (markers → ANSI inverse; single line;
     truncate to terminal width or 120).
   - Footer: `<shown> of <total> hits` + hint
     `replay: blackbox replay <id> --seek <seq>`.
   - `--session` resolves via `resolveSessionRef` (exit 3/4 semantics).
   - `--json` → `{total, hits}` with markers replaced by
     `«` / `»` (stable, terminal-independent).
   - Zero hits → `no hits` on stderr, exit 0, empty table suppressed.
4. Escaping torture cases handled by tests: `"`, `'`, `%`, `_`, `\`,
   `(`, `-`, `:` in queries (FTS quoting + LIKE ESCAPE both paths).

## Acceptance Criteria

- [ ] Unit: buildFtsQuery branches (multi-term AND, quoting, short-term →
      like, mixed-length → like, empty).
- [ ] Integration over imported fixtures: term unique to a fixture diff is
      found with `type=file_change` filter; `--tool Bash` narrows; `--path`
      matches the edited fixture path; since/until exclude by ts; offset
      pages without overlap; total stable across pages.
- [ ] Japanese: 3+ char Japanese term from fixtures matches via FTS; 2-char
      Japanese term matches via LIKE fallback; snippet contains the marked
      term (or the documented hand-rolled fallback is active + DESIGN
      updated).
- [ ] `text_truncated` events flagged in hits (`textTruncated: true` present
      for >64 KiB fixture event).
- [ ] SQL-injection probe: query `'; DROP TABLE sessions;--` returns
      normally (bound params only).
- [ ] Lint/typecheck/CI green.

## Validation

Local: `blackbox search <known real term>` and a Japanese query over real
data; attach hit counts + timing (`time blackbox search …`) to the PR
(informal; hard budgets in issue 23).

## Dependencies

- 09 (populated FTS), 11 (resolveSessionRef)

## Non-goals

- No search UI (18), no API endpoint (14).
- No operators (OR/NOT/phrase), no regex search (v2).
- No index tuning beyond ADR-004 caps (issue 23 measures).

## Design References

- DESIGN §15 (all), §16.4, §29 unknown #5
- ADR-004
