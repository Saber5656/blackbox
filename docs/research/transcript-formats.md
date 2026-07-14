# Research: Agent Transcript Formats (Claude Code / Codex CLI)

Date: 2026-07-05
Method: structural sampling of real transcript files on the development machine
(`jq` key extraction only — no content values were copied into this document).
Confidence: **observed** = verified on real files on 2026-07-05; **reported** =
known from public documentation/ecosystem knowledge, must be re-verified against
fixtures during implementation (see issue 05).

This research materially drives the adapter design in DESIGN.md §11 and issues
05, 06, 07, 08.

---

## 1. Claude Code session transcripts

### 1.1 Location (observed)

```
~/.claude/projects/<project-slug>/<session-uuid>.jsonl
```

- `<project-slug>` is the absolute project path with `/` and `.` replaced by `-`
  (e.g. `-Users-takagiyasushi-dev-blackbox`).
- One JSONL file per session. The filename stem is the session UUID.
- Sub-agent (sidechain) content may appear inline with `isSidechain: true`
  (observed field) and/or in separate files (reported — verify in issue 06).

### 1.2 Record envelope (observed)

Every line is a standalone JSON object. Observed `type` values and their key
sets:

| `type` | Observed keys |
|---|---|
| `user` | `uuid`, `parentUuid`, `sessionId`, `timestamp`, `cwd`, `gitBranch`, `version`, `isSidechain`, `userType`, `entrypoint`, `type`, `message`, and optionally `origin`, `permissionMode`, `promptId`, `promptSource`, `toolUseResult`, `sourceToolAssistantUUID` |
| `assistant` | same envelope plus `requestId`, `message` |
| `attachment` | envelope plus `attachment` |
| `last-prompt` | `lastPrompt`, `leafUuid`, `sessionId`, `type` |
| `queue-operation` | `operation`, `sessionId`, `timestamp`, `type`, optional `content` |

Reported (not present in the sampled file, must be tolerated): `summary`
(`{type:"summary", summary, leafUuid}`), `system`, `progress`,
`file-history-snapshot`. Unknown types MUST map to the normalized `raw` event,
never crash the parser.

### 1.3 Message content (observed)

- `assistant.message.content` is an array of blocks. Observed block types:
  `text`, `thinking`, `tool_use`. `tool_use` blocks carry `{id, name, input}`.
- `assistant.message.usage` observed keys: `input_tokens`, `output_tokens`,
  `cache_creation_input_tokens`, `cache_read_input_tokens`, plus extra keys
  (`cache_creation`, `inference_geo`, `iterations`, `server_tool_use`,
  `service_tier`, `speed`) that must be ignored gracefully.
- `assistant.message.model` (reported) carries the model id.
- `user.message.content` is either a plain string (human prompt) or an array
  containing `tool_result` blocks (`{type:"tool_result", tool_use_id, content,
  is_error?}`).
- Tool-result `user` lines carry a sidecar `toolUseResult` object whose shape
  varies by tool. Observed variants:
  - file tools: `{file, type}` (type is e.g. `create`/`update` — verify)
  - Bash: `{stdout, stderr, interrupted, isImage, noOutputExpected}`
- Multiple assistant lines may share one `requestId` (one line per content
  block). Token usage is attached per line and repeats per request — session
  totals must dedupe by `requestId` (see DESIGN.md §11.2).

### 1.4 File-modifying tools (reported — verify in issues 06/08)

- `Edit` input: `{file_path, old_string, new_string, replace_all?}`
- `Write` input: `{file_path, content}`
- `MultiEdit` input: `{file_path, edits: [{old_string, new_string}...]}`
- `NotebookEdit` input: `{notebook_path, ...}`
- `toolUseResult` for Edit historically includes `structuredPatch` /
  `originalFile` / `oldString` / `newString` — presence must be
  feature-detected, never assumed.

---

## 2. Codex CLI rollout files

### 2.1 Location (observed)

```
~/.codex/sessions/YYYY/MM/DD/rollout-YYYY-MM-DDThh-mm-ss-<uuid>.jsonl
```

### 2.2 Record envelope (observed)

Every line: `{"timestamp": "...", "type": "...", "payload": {...}}`.

Observed `type` values:

| `type` | `payload.type` | Observed payload keys |
|---|---|---|
| `session_meta` | — | `id`, `timestamp`, `cwd`, `originator`, `cli_version`, `source`, `thread_source`, `model_provider`, `base_instructions` |
| `turn_context` | — | `approval_policy`, `collaboration_mode`, `current_date`, `cwd`, `effort`, `model`, `permission_profile`, `personality`, `realtime_active`, `sandbox_policy`, `summary`, `timezone`, `turn_id` |
| `response_item` | `message` | `role`, `content`, optional `phase` |
| `response_item` | `reasoning` | `summary`, `content`, `encrypted_content` |
| `response_item` | `function_call` | `name`, `arguments` (JSON string), `call_id` |
| `response_item` | `function_call_output` | `call_id`, `output` |
| `event_msg` | `user_message` | `message`, `images`, `local_images`, `text_elements` |
| `event_msg` | `agent_message` | `message`, `phase`, `memory_citation` |
| `event_msg` | `token_count` | `info`, `rate_limits` |
| `event_msg` | `task_started` | `turn_id`, `model_context_window`, `collaboration_mode_kind`, `started_at` |
| `event_msg` | `task_complete` | `turn_id`, `duration_ms`, `last_agent_message`, `completed_at`, `time_to_first_token_ms` |

Reported (must be tolerated): `compacted`, `response_item` payload types
`local_shell_call`, `custom_tool_call`, `custom_tool_call_output`,
`web_search_call`; `event_msg` payload types `agent_reasoning`, `error`,
`turn_aborted`, and others.

### 2.3 Duplication hazard (observed)

User and assistant text appear **twice**: once as `response_item`
(`payload.type == "message"` with `role`) and once as `event_msg`
(`user_message` / `agent_message`). The adapter must pick exactly one stream
as canonical to avoid double events. Decision (DESIGN.md §11.3):
`response_item` is canonical; `event_msg` is consumed only for `token_count`
(and tolerated otherwise). Issue 07 includes a validation step comparing both
streams on real local data before finalizing.

### 2.4 Tool calls (observed + reported)

- `function_call.arguments` is a JSON-encoded string, e.g.
  `{"command":["bash","-lc","..."]}` for shell.
- Patches are applied via an `apply_patch` tool call (function name
  `apply_patch`, or a `shell` call whose command invokes `apply_patch` with a
  heredoc). The patch body uses the envelope grammar:

```
*** Begin Patch
*** Add File: <path>
+<line>
*** Update File: <path>
[*** Move to: <new path>]
@@ <context header>
 <context line> / +<added> / -<removed>
*** Delete File: <path>
*** End Patch
```

- `token_count.info` contains `total_token_usage` with keys
  `input_tokens`, `cached_input_tokens`, `output_tokens`, `total_tokens`
  (reported — verify exact nesting in issue 07). Values are cumulative for the
  session, so "last wins" for session totals.

### 2.5 Reasoning content

`reasoning` payloads may contain only `encrypted_content` (opaque). The
normalized `thinking` event must tolerate empty visible text and flag
`encrypted: true`.

---

## 3. Prior art (brief)

| Tool | Relevance | Takeaway |
|---|---|---|
| `claude-code-log` (OSS) | Converts Claude Code JSONL to HTML transcripts | Validates JSONL-file ingestion approach; no cross-session search/replay |
| LangSmith / Langfuse | Hosted LLM tracing | Server + SDK instrumentation model; too heavy for local-first v1, informs v2 |
| `asciinema` | Terminal session record/replay/share | Good UX model for replay (timing capped playback) and single-artifact sharing |

Conclusion: no existing local-first tool unifies Claude Code + Codex transcripts
with cross-session search, timeline replay, and redacted sharing. That is the
v1 gap blackbox targets.

---

## 4. Stability assessment & mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Either format adds new record types | High (both tools ship weekly) | Tolerant parsers: unknown → `raw` event; parse errors counted, never fatal |
| Field renames/removals | Medium | Feature-detect fields; fixtures pinned per observed version; adapter unit tests |
| Both streams present in Codex diverge (event_msg vs response_item) | Medium | Canonical-stream rule + count validation on real data (issue 07) |
| Sidechain/sub-agent file layout differs | Medium | native session id = filename stem; duplicate-session guard (issue 06) |

All schema claims marked **reported** above MUST be re-verified against real
local files during issue 05 (fixture authoring) before adapter implementation.
