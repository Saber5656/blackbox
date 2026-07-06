# ADR-004: SQLite FTS5 with trigram tokenizer for search

Date: 2026-07-05
Status: Accepted

## Context

Headline feature: search prompts, tool calls, and diffs across all recorded
sessions. Requirements:

- Substring-ish matching over code identifiers (`getUserById`, `--force`,
  file paths) — classic word tokenizers split these poorly.
- Japanese text must be searchable (the primary user writes prompts in
  Japanese). The default FTS5 `unicode61` tokenizer does not segment CJK, so
  whole-sentence tokens make Japanese queries useless.
- Zero external services (ADR-001) → SQLite-embedded search only.

Options:

- **A. FTS5 `unicode61`**: fast, small index; fails CJK and substring matching.
- **B. FTS5 `trigram`**: indexes every 3-byte-char window; substring and CJK
  matching work; index is larger (~3-5x text size) and queries need >= 3
  chars.
- **C. External tokenizer (lindera/mecab wasm)**: best Japanese relevance;
  heavy dependency, custom build complexity — over-budget for v1.

## Decision

**B. FTS5 `trigram` tokenizer**, one FTS table over normalized event text
(`text`, `tool_name`, `file_path` columns), kept in sync with the `events`
table by triggers.

- Queries shorter than 3 characters fall back to a bounded `LIKE '%q%'` scan
  with the same filters (slower; result cap enforced).
- Per-event indexed text is capped (`fts.maxIndexedTextBytes`, default 64 KiB)
  to bound index growth on huge diffs; the cap is recorded per event so the UI
  can show "search covers first 64 KiB" notices.
- Case folding: trigram tokenizer option `case_sensitive 0`.

## Consequences

- Substring search works for code tokens and Japanese without extra deps.
- DB size grows meaningfully (measured and budgeted in issue 23; mitigations:
  text cap, optional future `detail=none`).
- Relevance ranking is crude (bm25 over trigrams). Acceptable for v1; semantic
  or morphological search is a v2 item (DESIGN.md §3.3).
- Requires SQLite >= 3.34; better-sqlite3 11.x bundles a newer SQLite —
  issue 03 adds a startup assertion with a clear error.
