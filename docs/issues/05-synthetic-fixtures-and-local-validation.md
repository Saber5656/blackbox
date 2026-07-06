# Title

Synthetic fixtures, fixture generator, and local-data validation harness

## Summary

Create the committed synthetic transcript fixtures for both sources
(`fixtures/claude-code/*.jsonl`, `fixtures/codex/*.jsonl`), the generator
script `scripts/make-fixtures.ts`, shared adapter contract tests, and the
`npm run validate:local` harness that dry-runs adapters over the real
`~/.claude/projects` and `~/.codex/sessions` trees (local only, never CI).

## Context

Adapters (issues 06/07) are fixture-driven. Real transcripts cannot be
committed to this public repository (DESIGN §24, §25.2), so fixtures are
synthetic but must faithfully mirror the observed schemas documented in
docs/research/transcript-formats.md. The local validation harness is the
acceptance gate proving adapters work on real data without committing any of
it.

## Scope

- `scripts/make-fixtures.ts` + npm script `fixtures:make`
- `fixtures/claude-code/` (3 files), `fixtures/codex/` (3 files)
- `src/adapters/types.ts` (SourceAdapter interface + AdapterContext base)
- `src/adapters/contract.test.ts` (shared contract suite, parameterized;
  runs against real adapters once 06/07 land — until then it exports the
  suite function and a trivial self-test)
- `src/cli/commands/validate-local.ts` registered as hidden command
  `blackbox validate-local` + npm script `validate:local`
- CI guard: grep step failing on real-looking secrets in `fixtures/`

## Detailed Requirements

1. `src/adapters/types.ts`: the exact `SourceAdapter` interface from DESIGN
   §11.1, plus `AdapterContext` base `{ absPath, nativeId, parseErrors:
   number, unknownTypes: Map<string, number> }` and
   `NormalizedEventDraft` re-export from model.
2. Fixture scenarios — Claude Code (`fixtures/claude-code/`):
   - `session-basic.jsonl`: user string prompt → assistant text (+usage with
     requestId) → user prompt → assistant with thinking + text. Japanese text
     in at least one prompt (e.g. `認証トークンの更新処理を直して`).
   - `session-tools.jsonl`: assistant tool_use Bash → user tool_result with
     `toolUseResult {stdout, stderr, interrupted:false, isImage:false}` →
     tool_use Edit (old_string/new_string) → tool_result with
     `toolUseResult {type:"update", filePath, structuredPatch:[…]}` →
     tool_use Write (create) → result; one tool_result with
     `is_error: true`; one `summary` record; one `isSidechain: true`
     assistant line; envelope fields per research §1.2 (uuid, parentUuid,
     sessionId, timestamp, cwd, gitBranch, version, requestId).
   - `session-edge.jsonl`: an unknown record type (`{"type":"future-x", …}`),
     one malformed JSON line (literally `{"type": "user", "message":` — cut
     off), an `attachment` record, a `queue-operation`, a tool output string
     > 64 KiB (repeat pattern, deterministic), a `MultiEdit` tool_use.
   - File names double as native session ids; content `sessionId` fields use
     fixed UUIDs (`00000000-0000-4000-8000-0000000000NN`).
3. Fixture scenarios — Codex (`fixtures/codex/`):
   - `rollout-basic.jsonl`: `session_meta` (id, cwd, cli_version, originator)
     → `turn_context` (model) → response_item message user (with an
     `<environment_context>`-prefixed meta message AND a real user message) →
     reasoning (summary array) → message assistant → `event_msg`
     token_count (info.total_token_usage) → `event_msg` user_message /
     agent_message duplicates (to prove skip logic) → task_complete.
   - `rollout-tools.jsonl`: function_call `shell` (arguments JSON string with
     command array) → function_call_output (JSON string `{output, metadata}`)
     → function_call `apply_patch` whose patch envelope contains Add File,
     Update File with `*** Move to:`, Delete File sections →
     function_call_output; a `local_shell_call`; a `web_search_call`.
   - `rollout-edge.jsonl`: unknown envelope type, unknown response_item
     payload type, reasoning with only `encrypted_content`, malformed JSON
     line, `compacted` record, >64 KiB function_call_output.
4. `scripts/make-fixtures.ts`: deterministic generator (fixed timestamps
   starting `2026-01-01T00:00:00.000Z`, +1s per record; fixed ids; no
   randomness, no Date.now) that writes exactly the files above. Committed
   fixtures must equal regenerated output (`git diff --exit-code` after
   `npm run fixtures:make` — CI step).
5. Contract test suite `adapterContractTests(adapter, fixtureDir)` asserting
   for every fixture file: parseLine never throws (malformed line → error
   event + parseErrors increment), unknown types → `raw` with
   `unknownTypes` counted, all drafts pass model validation (type guards),
   ts monotonic non-decreasing OR null, and `sessionPatch` returns
   started/ended timestamps.
6. `blackbox validate-local` (hidden from help):
   - Iterates BOTH real roots from config; for each file: stream lines,
     run the registered adapter's parseLine into a counting sink (NO DB
     writes). Uses adapters registered in a `src/adapters/registry.ts`
     (this issue creates the registry with an empty map; 06/07 register).
   - Output table per source: files, lines, events by type, parse errors,
     unknown record types with counts (top 20), duration. `--json` flag.
   - Exit 0 even with parse errors (it is a report, not a gate — the gate is
     human review of the report in issues 06/07).
   - Refuses to run when both adapters are unregistered: message
     "no adapters registered yet".
7. CI fixture guard: workflow step
   `grep -rInE '(sk-ant-|sk-proj-|AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{36})' fixtures/ && exit 1 || true`
   style (invert logic properly) ensuring no real-looking keys in fixtures;
   plus the regeneration diff check from req 4.

## Acceptance Criteria

- [ ] `npm run fixtures:make` regenerates byte-identical committed fixtures.
- [ ] Fixture files are valid JSONL except the two deliberately malformed
      lines; every scenario listed above is present (checklist in PR
      description mapping scenario → file:line).
- [ ] Contract suite exported and self-tested (runs against a stub adapter).
- [ ] `blackbox validate-local` prints the "no adapters registered" message
      and exits 1 at this stage.
- [ ] CI includes fixture-regeneration check and secret-pattern guard; both
      green.
- [ ] No string in `fixtures/` matches the secret-guard patterns; no content
      copied from real transcripts (reviewer attestation required in PR).

## Validation

1. `npm run fixtures:make && git diff --exit-code fixtures/`
2. `node dist/cli/index.js validate-local` → registered-adapter error.
3. Manual review: compare each fixture record's key set against the tables in
   docs/research/transcript-formats.md §1.2/§2.2 — every observed key present.

## Dependencies

- 01 (scaffold), 02 (config roots), 04 (model types for drafts)

## Non-goals

- No real adapter implementations (06/07).
- No DB writes from validate-local, ever.
- No fixture for every historical format version — current observed schema
  only.

## Design References

- DESIGN §11.1 (interface), §24 (fixture policy), §25.2 (fixtures,
  validate:local), §28–29 (risks/unknowns)
- docs/research/transcript-formats.md (all sections)
