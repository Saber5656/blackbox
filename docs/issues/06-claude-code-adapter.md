# Title

Claude Code transcript adapter

## Summary

Implement `src/adapters/claude-code.ts`: a pure, tolerant, per-line parser
converting Claude Code JSONL records into normalized event drafts exactly per
the mapping table in DESIGN §11.2, including sidechain flags, title
resolution, and requestId-deduplicated token totals. Register it in the
adapter registry so `validate-local` covers it.

## Context

This is the first real source. The observed record shapes are documented in
docs/research/transcript-formats.md §1 (observed 2026-07-05); anything marked
"reported" must be re-verified against real local data via
`blackbox validate-local` before this issue closes.

## Scope

- `src/adapters/claude-code.ts` + registration in `src/adapters/registry.ts`
- `src/adapters/claude-code.test.ts` (fixture-driven) + contract suite hookup

## Detailed Requirements

1. Adapter surface (DESIGN §11.1): `name: 'claude-code'`,
   `filePatterns: ['*/*.jsonl']`, `sessionKeyForFile(absPath) →` filename
   stem (strip `.jsonl`); return null for non-`.jsonl`.
2. Context state: `firstLineSeen`, `sessionMetaEmitted`, `titleFromSummary`,
   `firstUserText`, `perRequestUsage: Map<requestId, Usage>` (last line per
   requestId wins), `lastTs`, `startTs`, `lastModel`, `cwd`, `gitBranch`,
   `agentVersion`, `entrypoint`, plus base fields from issue 05.
3. `parseLine` mapping — implement DESIGN §11.2 table exactly. Key details:
   - Emit `session_meta` once, on the first successfully parsed line, from
     that line's envelope (cwd, gitBranch, version→agentVersion, entrypoint);
     put the in-file `sessionId` into `payload.extra.sessionId`.
   - `user` with string content → one `user_message`
     (`meta: envelope.isMeta === true`).
   - `user` with array content → per block, in array order:
     `text` → `user_message`; `tool_result` → `tool_result` draft with
     `toolUseId = block.tool_use_id`, `output` = flatten(block.content)
     (string passes through; array of `{type:'text',text}` joins with `\n`;
     non-text blocks → `[<type> block]` placeholder),
     `isError = block.is_error === true`, `detail = envelope.toolUseResult`
     (verbatim, may be huge — offload handles it downstream).
   - `assistant` → per content block in order: `text` → `assistant_message`
     (`model` from `message.model`); `thinking` → `thinking` (field name
     `thinking` OR `text` inside the block — read whichever exists);
     `tool_use` → `tool_call {toolName: block.name, toolUseId: block.id,
     input: block.input}`; set `filePath` when input has a string `file_path`
     or `notebook_path`.
   - `summary` → `summary {kind:'title', text: rec.summary}`; remember as
     title candidate (last one wins).
   - `system` → if `rec.subtype` contains `error` (case-insensitive) or
     `rec.isApiErrorMessage === true` → `error {origin:'agent'}` with best
     text (`rec.content` string or JSON), else `raw`.
   - `attachment`, `queue-operation`, `last-prompt`,
     `file-history-snapshot`, `progress`, anything else → `raw
     {sourceType: rec.type, json: rec}` (count in `unknownTypes` only when
     not in the known list above).
   - Envelope `isSidechain === true` → `sidechain: true` on every event from
     that line.
   - `ts` = envelope `timestamp` (pass through; do not reformat); track
     start/last.
   - Malformed JSON → `[{type:'error', payload:{origin:'parser', text:
     'invalid JSON at line N: <err.message>'}}]`, `parseErrors++`, `ts` null.
   - Usage: on every assistant line with `message.usage` and `requestId`,
     store into `perRequestUsage` (overwrite → last wins).
4. `sessionPatch(ctx)` returns: `startedAt`, `endedAt`, `projectDir: cwd`,
   `gitBranch`, `model: lastModel`, `agentVersion`,
   `title = titleFromSummary ?? firstUserText?.slice-to-80-chars`
   (newlines→space, trim; skip meta user messages),
   `inputTokens` = final request's `input_tokens +
   cache_read_input_tokens + cache_creation_input_tokens` (missing → treat
   as 0; "final" = the request of the last assistant line carrying usage),
   `outputTokens` = Σ over perRequestUsage values of `output_tokens`.
5. Tolerance rules: every field access defensive (`typeof` checks); absent
   `message.content` → treat as empty; unknown content block types inside
   messages → `raw` event per block with `sourceType:
   'claude-code:block:<type>'`. Never throw.

## Acceptance Criteria

- [ ] Contract suite (issue 05) passes for all three claude-code fixtures.
- [ ] Snapshot test: `session-tools.jsonl` → exact ordered list of
      normalized drafts (types/roles/toolUseIds/filePaths asserted
      literally in the test, not just snapshotted).
- [ ] `session-basic.jsonl`: token math asserted (construct fixture so
      expected input/output totals are hand-computable); title from first
      user message; Japanese text survives byte-identically.
- [ ] `session-edge.jsonl`: malformed line → 1 parser error event + count;
      unknown type → raw + unknownTypes entry; >64 KiB output draft carries
      full string (offload happens downstream, not in the adapter).
- [ ] Sidechain flag propagates to all drafts of a flagged line.
- [ ] `validate-local` run on the real `~/.claude/projects` tree completes
      with **zero crashes**; the PR description includes the report output
      (counts only — no content), and every unknown record type listed is
      either added to the known-`raw` list or mapped, before merge.
- [ ] Lint/typecheck/CI green.

## Validation

1. `npm test -- adapters/claude-code`
2. `npm run build && node dist/cli/index.js validate-local --json`
   (local only) → attach summarized counts to the PR; reviewer confirms
   unknown-type handling decision for each listed type.

## Dependencies

- 04 (model), 05 (fixtures, contract suite, registry, validate-local)

## Non-goals

- No file_change derivation (issue 08 consumes this adapter's tool events).
- No DB access, no file I/O in the adapter (importer's job).
- No handling of separate sub-agent transcript files beyond importing them
  as their own sessions (DESIGN §29 unknown #3).

## Design References

- DESIGN §8, §11.1, §11.2
- docs/research/transcript-formats.md §1, §4
