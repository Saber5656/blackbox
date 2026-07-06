# Title

Codex CLI rollout adapter

## Summary

Implement `src/adapters/codex.ts`: parse Codex rollout JSONL
(`{timestamp, type, payload}` envelopes) into normalized event drafts per the
mapping table in DESIGN §11.3, resolving the response_item/event_msg
duplication by treating `response_item` as canonical, and taking session token
totals from the LAST `token_count` snapshot. Includes a mandatory validation
step on real local data to confirm the canonical-stream decision.

## Context

Research (docs/research/transcript-formats.md §2) confirmed user/assistant
text appears in both `response_item` and `event_msg` streams. DESIGN §11.3
picks `response_item` as canonical; §29 unknown #2 flags `codex exec`
(non-interactive) sessions as the risk case — this issue must verify against
real data and either confirm or update DESIGN before closing.

## Scope

- `src/adapters/codex.ts` + registry registration
- `src/adapters/codex.test.ts` + contract suite hookup

## Detailed Requirements

1. Adapter surface: `name: 'codex'`,
   `filePatterns: ['**/rollout-*.jsonl']`,
   `sessionKeyForFile(absPath, firstLine)`: parse firstLine; if it is a
   `session_meta` with `payload.id` string → that id; else filename stem
   minus `rollout-` prefix; non-matching filename → null.
2. Context state: `nativeId`, `startTs`, `lastTs`, `lastModel`,
   `firstUserText`, `lastTokenUsage`, `cwd`, `agentVersion`, `entrypoint`,
   plus issue-05 base fields.
3. `parseLine` mapping — implement DESIGN §11.3 table exactly. Key details:
   - Envelope: `{timestamp, type, payload}`; event `ts` = envelope
     timestamp; missing/invalid envelope fields → tolerate (ts null).
   - `session_meta` → `session_meta` draft
     `{source:'codex', nativeId: payload.id, cwd: payload.cwd,
     agentVersion: payload.cli_version, entrypoint: payload.originator,
     extra: {source: payload.source, threadSource: payload.thread_source,
     modelProvider: payload.model_provider}}`. Do NOT copy
     `base_instructions` into extra (large; it stays only in raw_ref-less
     oblivion — explicitly dropped, documented in a code comment).
   - `turn_context` → `raw` draft; update `lastModel = payload.model` when
     string.
   - `response_item`, `payload.type == 'message'`:
     text = join content items (`input_text`/`output_text`/`text` fields)
     with `\n`; role user → `user_message` with `meta: true` when text
     trim-starts with `<environment_context>` or `<user_instructions>`;
     role assistant → `assistant_message {model: lastModel}`; other roles →
     `raw`.
   - `payload.type == 'reasoning'` → `thinking`: text = payload.summary[]
     `.text` fields joined `\n\n` (fallback: payload.content[] same way);
     `encrypted: true` when no visible text but `encrypted_content` present.
   - `payload.type == 'function_call'` → `tool_call {toolName: payload.name,
     toolUseId: payload.call_id, input: tryParseJSON(payload.arguments)}`;
     `displayCommand`: when input has `command` array → `command.join(' ')`;
     `filePath` never set here (patch paths handled by issue 08).
   - `payload.type == 'local_shell_call'` → `tool_call {toolName:'shell',
     toolUseId: payload.call_id ?? null, input: payload.action,
     displayCommand: action.command?.join(' ')}`.
   - `payload.type == 'function_call_output'` → `tool_result
     {toolUseId: payload.call_id, output, isError}` where: if
     `payload.output` parses as JSON object with string `output` → use inner
     output, `detail` = remaining keys, `isError` = detail.metadata?
     .exit_code not in {0, undefined}; else output = String(payload.output),
     isError = false.
   - `payload.type == 'custom_tool_call'` → `tool_call {toolName:
     payload.name ?? 'custom', toolUseId: payload.call_id, input: payload}`;
     `custom_tool_call_output` → like function_call_output;
     `web_search_call` → `tool_call {toolName:'web_search', input: payload}`
     (no result pairing expected).
   - `event_msg`: `token_count` → `token_usage` draft from
     `payload.info.total_token_usage` mapping keys `input_tokens→
     inputTokens, cached_input_tokens→cacheReadTokens, output_tokens→
     outputTokens, total_tokens→totalTokens` (feature-detect nesting: if
     `info.total_token_usage` absent, try `info` itself); remember as
     `lastTokenUsage`. `user_message`/`agent_message` → return `[]`
     (skipped; increment a `skippedDuplicates` counter on ctx).
     All other payload types → `raw`.
   - `compacted` → `summary {kind:'compaction', text: payload.message ??
     'context compacted'}`.
   - Unknown envelope type → `raw` + unknownTypes count. Malformed JSON →
     parser `error` draft (same rule as issue 06).
4. `sessionPatch`: `startedAt` = session_meta ts (fallback first ts),
   `endedAt = lastTs`, `projectDir = cwd`, `model = lastModel`,
   `agentVersion`, `title = firstUserText` (non-meta, 80 chars,
   newline-collapsed), tokens from `lastTokenUsage`
   (`inputTokens, outputTokens`; cumulative → last wins, DESIGN §11.3).
5. Tolerance identical to issue 06 (never throw; defensive access).

## Acceptance Criteria

- [ ] Contract suite passes for all three codex fixtures.
- [ ] `rollout-basic.jsonl` test asserts: exactly one `user_message` with
      `meta:false` and one with `meta:true`; NO events from `event_msg`
      user/agent duplicates (skippedDuplicates == 2); token totals equal the
      fixture's last token_count snapshot; title from the real user message.
- [ ] `rollout-tools.jsonl` test asserts literal ordered drafts: shell
      tool_call with displayCommand, paired function_call_output via call_id,
      apply_patch tool_call preserved verbatim (input contains the patch
      envelope string), local_shell_call and web_search_call mapped.
- [ ] `rollout-edge.jsonl`: encrypted-only reasoning → thinking with
      `encrypted:true` and empty text; compacted → summary; unknown types →
      raw; malformed line counted.
- [ ] **Canonical-stream validation**: `validate-local` on the real
      `~/.codex/sessions` tree completes with zero crashes; PR includes the
      counts report AND a comparison count of `response_item` messages vs
      skipped `event_msg` duplicates. If any real session yields
      `user_message` count 0 while skippedDuplicates > 0 (the `codex exec`
      risk), STOP and update DESIGN §11.3 + this issue before merging
      (docs-first rule).
- [ ] Lint/typecheck/CI green.

## Validation

1. `npm test -- adapters/codex`
2. `node dist/cli/index.js validate-local --json` (local) → attach counts to
   PR; reviewer signs off on the canonical-stream evidence explicitly.

## Dependencies

- 04 (model), 05 (fixtures, contract suite, registry)

## Non-goals

- No patch parsing (issue 08).
- No handling of `~/.codex/archived_sessions` or other trees (v2).
- No DB/file I/O inside the adapter.

## Design References

- DESIGN §8, §11.1, §11.3, §29 (unknown #2)
- docs/research/transcript-formats.md §2, §4
