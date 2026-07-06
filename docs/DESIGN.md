# blackbox — Product Design (v1)

Status: Canonical design for v1. All implementation issues derive from this
document. If an issue and this document disagree, this document wins and must
be updated first.

Related documents:
- docs/ISSUE_PLAN.md — execution map
- docs/issues/*.md — per-issue implementation drafts
- docs/decisions/ADR-001..004 — architecture decisions
- docs/research/transcript-formats.md — source format research (observed facts)

---

## 1. Product overview

**blackbox** is a local-first recorder, replayer, search engine, and sharing
tool for AI coding-agent runs. It ingests the session transcripts that agents
already write to disk (v1: Claude Code, Codex CLI), normalizes them into a
single event model, indexes them in SQLite, and lets the user:

- **Record**: import all historical and ongoing runs incrementally (`sync`,
  `watch`).
- **Browse & search**: full-text search across prompts, assistant output,
  thinking, tool calls, tool results, and file diffs, with filters
  (`search`, web UI).
- **Replay**: step through a run event-by-event in the terminal or the web
  viewer, with recorded timing (capped) — re-presentation, never re-execution.
- **Share**: export a run as a portable `.bbox` bundle (re-importable) or a
  self-contained single-file HTML viewer, with secret redaction on by default.

### 1.1 Why (product rationale)

Agent runs are today's equivalent of CI logs: they contain the *why* behind
code changes (prompts, decisions, failed attempts, diffs) but they are trapped
in per-tool, per-machine JSONL files with no search, no unified viewer, and no
safe way to share. blackbox is the flight recorder for that history.

### 1.2 North-star user stories (v1 must satisfy all)

| # | Story |
|---|---|
| US-1 | "Import everything my agents have ever done on this machine with one command." (`blackbox sync`) |
| US-2 | "Which session touched `auth/token.ts`? What prompt led to that diff?" (`blackbox search`) |
| US-3 | "Show me exactly what the agent did in that session, in order." (`blackbox replay`, web timeline) |
| US-4 | "Keep recording while I work." (`blackbox watch`) |
| US-5 | "Send this run to a teammate who does not have blackbox, without leaking my API keys." (HTML export + redaction) |
| US-6 | "Move a run between machines / archive it." (`.bbox` export/import) |

## 2. Personas

- **P1 — Agent power user (primary)**: runs Claude Code and Codex daily across
  many repos; wants history, search, and post-mortems. Japanese and English
  prompts mixed.
- **P2 — Teammate/reviewer (secondary)**: receives a shared run; has no
  blackbox installed; opens a single HTML file in a browser.

## 3. Scope

### 3.1 v1 scope

1. Sources: Claude Code transcripts, Codex CLI rollouts (read-only ingestion).
2. Normalized event model + SQLite storage + content-addressed blob store.
3. Incremental sync + polling watch mode.
4. CLI: `sync`, `watch`, `list`, `show`, `delete`, `search`, `replay`,
   `export`, `import`, `ui`, `--version`, `--help`.
5. Full-text search (FTS5 trigram; CJK + substring capable) with filters.
6. Terminal replay.
7. Local web viewer (session list, timeline, replay, search).
8. Exports: `.bbox` (zip), single-file HTML, Markdown, JSON; `.bbox` import.
9. Redaction engine (built-in secret patterns + user patterns), default-on for
   all exports.
10. Tests (unit/integration), CI (macOS+Linux), user documentation.

### 3.2 v1 non-goals

- Re-execution of recorded tool calls ("replay" is display-only).
- Live hook/OTel instrumentation of agents (ADR-002).
- Sources other than the two above (no Gemini CLI, Cursor, Aider...).
- Multi-user, server deployment, auth, cloud sync, team workspaces.
- Editing/annotating transcripts.
- Windows support (best-effort only; not tested in CI).
- npm publish (install via `npm install -g .` / `npm link`).
- Deriving file changes from raw shell commands (e.g. `cat > f << EOF`), and
  Codex `unified_exec`-style exotic tools — only structured edit tools listed
  in §12 are extracted.

### 3.3 v2 deferred (recorded for continuity; NOT planned issues)

Live hook capture; more agent sources; annotations & comments; team server
mode reusing `.bbox` as wire format; cost/usage analytics dashboards; semantic
search (embeddings) and morphological Japanese tokenization; re-run/fork a
session from step N; TUI (ink) browser; blob GC command; encryption at rest;
Playwright browser e2e; npm publish; Windows CI.

## 4. Architecture overview

```
                ~/.claude/projects/**/*.jsonl        ~/.codex/sessions/**/rollout-*.jsonl
                          │                                     │
                          ▼                                     ▼
               ┌─ adapters/claude-code ─┐            ┌─ adapters/codex ─┐
               │  parseLine() → events  │            │  parseLine() ... │
               └───────────┬────────────┘            └────────┬─────────┘
                           └────────────┬─────────────────────┘
                                        ▼
                          ingest/importer (incremental, offset-resume,
                          file-change extraction, blob offload, tx batches)
                                        │
                                        ▼
        ~/.blackbox/blackbox.db (SQLite WAL: sessions/events/FTS5/source_files)
        ~/.blackbox/blobs/<sha256>     (content-addressed large payloads)
                                        │
        ┌───────────────┬───────────────┼────────────────────┬──────────────┐
        ▼               ▼               ▼                    ▼              ▼
   CLI list/show   CLI search      CLI replay          export engine    HTTP API (express)
        │               │               │              (redact → bbox/       │
        └───────────────┴───────────────┘               html/md/json)        ▼
                                                            ▲       viewer SPA (React)
                                                            │       list/timeline/replay/search
                                                    single-file HTML ◄─ embedded data island
```

Module boundaries are physical directories (§6); dependencies point downward
only: `cli`/`server`/`export` → `query`/`ingest` → `adapters`/`model` →
`db`/`config`/`util`. The viewer depends only on the HTTP API JSON contracts
(§18) or the embedded data island (§20.3).

## 5. Technology stack

See ADR-003 for the pinned stack (TypeScript strict ESM, Node >= 20,
commander, better-sqlite3, express, React+Vite, vitest). No dependency may be
added beyond ADR-003's table without updating that ADR.

## 6. Repository layout

```
blackbox/
├── package.json              # name "blackbox", private:true, bin: {blackbox, bbx}
├── tsconfig.json             # strict, ESM, NodeNext
├── vitest.config.ts
├── eslint.config.js
├── .prettierrc.json
├── .github/workflows/ci.yml
├── docs/                     # this design + issues (already present)
├── fixtures/                 # committed SYNTHETIC transcripts only (§25.2)
│   ├── claude-code/*.jsonl
│   └── codex/*.jsonl
├── scripts/
│   └── make-fixtures.ts      # synthetic fixture generator (issue 05)
├── src/
│   ├── cli/
│   │   ├── index.ts          # commander program, global flags, error handler
│   │   └── commands/{sync,watch,list,show,delete,search,replay,export,import,ui}.ts
│   ├── config/config.ts      # load/merge/validate config (§7)
│   ├── db/
│   │   ├── database.ts       # open, pragmas, migration runner, version assert
│   │   └── migrations/0001_init.sql
│   ├── model/
│   │   ├── events.ts         # normalized types (§8), type guards
│   │   └── normalize.ts      # payload→row mapping, text derivation, blob offload
│   ├── blob/store.ts         # content-addressed store (§10)
│   ├── adapters/
│   │   ├── types.ts          # SourceAdapter interface (§11.1)
│   │   ├── claude-code.ts
│   │   └── codex.ts
│   ├── ingest/
│   │   ├── scanner.ts        # discover source files
│   │   ├── importer.ts       # incremental engine (§13)
│   │   └── filechange.ts     # diff extraction (§12)
│   ├── query/
│   │   ├── sessions.ts       # list/get/stats/delete/resolve-prefix
│   │   └── search.ts         # FTS query layer (§15)
│   ├── replay/render.ts      # terminal event renderer + player (§17)
│   ├── export/
│   │   ├── redact.ts         # §21
│   │   ├── bbox.ts html.ts markdown.ts json.ts
│   ├── server/
│   │   ├── app.ts            # express app factory (no listen)
│   │   └── routes.ts         # §18 endpoints
│   └── util/{ids,time,hash,log,table,paths}.ts
├── viewer/                   # Vite React app (built → dist/viewer, dist/viewer-embed)
│   ├── index.html
│   ├── vite.config.ts        # normal build
│   ├── vite.embed.config.ts  # single-file build (vite-plugin-singlefile)
│   └── src/
│       ├── main.tsx App.tsx routes.tsx
│       ├── data/provider.ts  # DataProvider interface: HttpProvider | EmbeddedProvider
│       ├── pages/{SessionList,Timeline,Replay,Search}.tsx
│       └── components/{EventCard,DiffView,Filters,...}.tsx
└── dist/                     # build output (gitignored)
```

## 7. Configuration & data directory

### 7.1 Resolution order (highest wins)

1. CLI flags (e.g. `--data-dir`, `--port`)
2. Environment: `BLACKBOX_HOME` (data dir), `BLACKBOX_DEBUG=1`
3. Config file `<dataDir>/config.json`
4. Built-in defaults

Data dir default: `~/.blackbox`. Created on first use (mode 0700) with
subdirs `blobs/`. The SQLite file is `<dataDir>/blackbox.db`.

### 7.2 config.json schema (all keys optional; unknown keys → warning, not error)

```jsonc
{
  "sources": {
    "claude-code": { "enabled": true, "roots": ["~/.claude/projects"] },
    "codex":       { "enabled": true, "roots": ["~/.codex/sessions"] }
  },
  "sync":      { "watchIntervalSeconds": 5 },
  "blob":      { "offloadThresholdBytes": 65536, "htmlInlineCapBytes": 262144 },
  "fts":       { "maxIndexedTextBytes": 65536 },
  "redaction": {
    "enabled": true,
    "customPatterns": [ { "name": "corp-token", "pattern": "CORP-[0-9]{8}", "flags": "g" } ]
  },
  "ui":        { "port": 7710 }
}
```

Validation: hand-rolled validator (no schema lib). Invalid values → startup
error naming the key, expected type, and received value. `~` in paths expands
to `os.homedir()`.

## 8. Normalized event model

One session = ordered list of events. TypeScript source of truth in
`src/model/events.ts`.

### 8.1 Event types and payloads

| type | role | payload (TypeScript-ish) | text column (FTS source) |
|---|---|---|---|
| `session_meta` | — | `{ source, nativeId, cwd?, gitBranch?, agentVersion?, entrypoint?, extra? }` | `''` |
| `user_message` | user | `{ text, meta?: boolean }` (`meta`: injected/system-expanded content) | text |
| `assistant_message` | assistant | `{ text, model? }` | text |
| `thinking` | assistant | `{ text, encrypted?: boolean }` | text |
| `tool_call` | assistant | `{ toolName, toolUseId, input: unknown, displayCommand? }` | toolName + display/input string |
| `tool_result` | tool | `{ toolUseId, output: string, isError: boolean, detail?: unknown }` | output |
| `file_change` | tool | `{ path, kind: 'create'\|'modify'\|'delete'\|'rename', renameFrom?, diff?: string, fragment?: boolean, additions?: number, deletions?: number, derivedFrom: toolUseId }` | path + diff |
| `summary` | — | `{ text, kind: 'title'\|'compaction' }` | text |
| `token_usage` | — | `{ inputTokens?, outputTokens?, cacheReadTokens?, cacheCreationTokens?, totalTokens? }` | `''` |
| `error` | — | `{ text, origin: 'agent'\|'parser' }` | text |
| `raw` | — | `{ sourceType: string, json: unknown }` | JSON string capped 4096 bytes |

Rules:
- `fragment: true` on `file_change` means the diff was reconstructed from an
  edit fragment (old/new strings), not a whole-file diff.
- Every event may carry `sidechain: true` (Claude sub-agent content) as a
  top-level payload flag; renderers show a badge and default filters keep them.
- `text` derivation is centralized in `model/normalize.ts::deriveText(event)`
  and capped at `fts.maxIndexedTextBytes` (§15.3); when capped, event row gets
  `text_truncated = 1`.

### 8.2 Event envelope (DB row / API JSON)

```ts
interface BlackboxEvent {
  sessionId: string;      // "<source>:<nativeId>"
  seq: number;            // 0-based, contiguous per session, import order
  ts: string | null;      // ISO 8601 UTC with ms
  type: EventType;
  role: 'user' | 'assistant' | 'tool' | null;
  toolName: string | null;
  toolUseId: string | null;
  filePath: string | null;   // for file_change and file-tool calls
  text: string;              // derived searchable text (capped)
  payload: unknown;          // per-type payload; large fields → {"$blob": {hash,size,preview}}
}
```

### 8.3 Session identity

- `session.id = "<source>:<nativeId>"`, e.g. `claude-code:0195c2f3-…` /
  `codex:0195…`.
- Claude Code `nativeId` = transcript filename stem (stable, unique; the
  in-file `sessionId` field is informative only — see research §1).
- Codex `nativeId` = `session_meta.payload.id`; fallback = filename stem.
- CLI accepts any unambiguous prefix of `nativeId` or full `id`
  (resolution in `query/sessions.ts::resolveSessionRef`, exit code 4 on
  ambiguity listing candidates).

## 9. SQLite schema (migration 0001)

```sql
PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA foreign_keys=ON;

CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);
-- meta rows: schema_version='1', format_version='1', created_at=ISO

CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,                -- 'claude-code' | 'codex'
  native_id TEXT NOT NULL,
  title TEXT,
  project_dir TEXT,
  git_branch TEXT,
  model TEXT,
  started_at TEXT, ended_at TEXT,
  event_count INTEGER NOT NULL DEFAULT 0,
  user_msg_count INTEGER NOT NULL DEFAULT 0,
  assistant_msg_count INTEGER NOT NULL DEFAULT 0,
  tool_call_count INTEGER NOT NULL DEFAULT 0,
  file_change_count INTEGER NOT NULL DEFAULT 0,
  input_tokens INTEGER, output_tokens INTEGER,
  imported_from TEXT,                  -- original .bbox filename when imported
  imported_at TEXT NOT NULL,
  UNIQUE(source, native_id)
);

CREATE TABLE events (
  id INTEGER PRIMARY KEY,              -- rowid
  session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  seq INTEGER NOT NULL,
  ts TEXT,
  type TEXT NOT NULL,
  role TEXT,
  tool_name TEXT,
  tool_use_id TEXT,
  file_path TEXT,
  text TEXT NOT NULL DEFAULT '',
  text_truncated INTEGER NOT NULL DEFAULT 0,
  payload TEXT NOT NULL,               -- JSON string
  UNIQUE(session_id, seq)
);
CREATE INDEX idx_events_session ON events(session_id, seq);
CREATE INDEX idx_events_type    ON events(type);
CREATE INDEX idx_events_tool    ON events(tool_name) WHERE tool_name IS NOT NULL;
CREATE INDEX idx_events_path    ON events(file_path) WHERE file_path IS NOT NULL;

CREATE TABLE blobs (
  hash TEXT PRIMARY KEY,               -- sha256 hex
  size INTEGER NOT NULL,
  created_at TEXT NOT NULL
);
CREATE TABLE event_blobs (
  event_id INTEGER NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  blob_hash TEXT NOT NULL REFERENCES blobs(hash),
  field TEXT NOT NULL,                 -- JSON pointer-ish, e.g. "payload.output"
  PRIMARY KEY (event_id, blob_hash, field)
);

CREATE TABLE source_files (
  id INTEGER PRIMARY KEY,
  source TEXT NOT NULL,
  path TEXT NOT NULL UNIQUE,           -- absolute
  session_id TEXT,                     -- filled once known
  bytes_ingested INTEGER NOT NULL DEFAULT 0,
  lines_ingested INTEGER NOT NULL DEFAULT 0,
  size_bytes INTEGER, mtime_ms INTEGER,
  first_line_hash TEXT,                -- sha256 of first line, rotation guard
  error_count INTEGER NOT NULL DEFAULT 0,
  ignored INTEGER NOT NULL DEFAULT 0,  -- set by delete: sync skips
  last_synced_at TEXT
);

CREATE VIRTUAL TABLE events_fts USING fts5(
  text, tool_name, file_path,
  content='events', content_rowid='id',
  tokenize="trigram case_sensitive 0"
);
CREATE TRIGGER events_ai AFTER INSERT ON events BEGIN
  INSERT INTO events_fts(rowid, text, tool_name, file_path)
  VALUES (new.id, new.text, new.tool_name, new.file_path);
END;
CREATE TRIGGER events_ad AFTER DELETE ON events BEGIN
  INSERT INTO events_fts(events_fts, rowid, text, tool_name, file_path)
  VALUES ('delete', old.id, old.text, old.tool_name, old.file_path);
END;
-- events are never UPDATEd in v1 (import is append-only; delete is whole-session)
```

Migration runner: `src/db/database.ts` applies `migrations/NNNN_*.sql` in
lexicographic order inside one transaction each, recording `schema_version`.
Startup asserts `sqlite_version() >= 3.34.0` (trigram) with a clear error.

## 10. Blob store

- Location: `<dataDir>/blobs/<first2>/<sha256hex>` (fan-out dir = first 2 hex
  chars). Content bytes as-is (UTF-8 text in practice).
- Write: hash → if file exists, done (dedupe); else write temp + rename
  (atomic). Insert `blobs` row if absent.
- Offload rule (in `model/normalize.ts`): any **string** payload field whose
  UTF-8 length > `blob.offloadThresholdBytes` (default 64 KiB) is replaced by
  `{"$blob": {"hash": "...", "size": n, "preview": "<first 1024 chars>"}}`
  and registered in `event_blobs` with its field path. `text` derivation runs
  BEFORE offload (search still covers capped text).
- Read: `blob/store.ts::readBlob(hash, {maxBytes?})`.
- Orphan cleanup happens only inside `delete` (§16.5): after deleting a
  session, delete `blobs` rows (and files) whose hash no longer appears in
  `event_blobs`.

## 11. Source adapters

### 11.1 Adapter interface (`src/adapters/types.ts`)

```ts
interface SourceAdapter {
  readonly name: 'claude-code' | 'codex';
  /** glob patterns relative to each configured root, e.g. ["**/*.jsonl"] */
  readonly filePatterns: string[];
  /** native session id for a file (filename/first line), null = not a session file */
  sessionKeyForFile(absPath: string, firstLine: string | null): string | null;
  /** create per-file parse state (tool-call pairing, usage accumulators) */
  createContext(file: { absPath: string; nativeId: string }): AdapterContext;
  /** parse one line → zero or more normalized events (never throws) */
  parseLine(line: string, ctx: AdapterContext): NormalizedEventDraft[];
  /** session-level fields learned so far (title, tokens, model, timestamps) */
  sessionPatch(ctx: AdapterContext): Partial<SessionFields>;
}
```

Contracts (enforced by shared adapter tests, issue 05/06/07):
- `parseLine` MUST NOT throw. Malformed JSON → return
  `[{type:'error', payload:{origin:'parser', text:'<reason>'}}]` and increment
  `ctx.parseErrors`. Unknown record type → one `raw` event.
- Adapters are pure per-file (no I/O, no DB); the importer owns files and
  transactions. This makes them trivially unit-testable on fixtures.
- `sessionPatch` is re-computed after every batch; importer merges non-null
  fields (last write wins) and recomputes counters from SQL.

### 11.2 Claude Code adapter (source facts: research §1)

- Roots: `~/.claude/projects`; `filePatterns: ["*/*.jsonl"]`.
- `nativeId` = filename stem (research §1.2 note; the in-file `sessionId` is
  recorded into `session_meta.extra` but not used as identity).
- Line mapping:

| Source line | Normalized events |
|---|---|
| first parsed line (any type) | `session_meta` (cwd, gitBranch, version, entrypoint) |
| `type:"user"`, `message.content` is string | `user_message` (`meta:true` if envelope `isMeta` true) |
| `type:"user"`, content array | per `text` block → `user_message`; per `tool_result` block → `tool_result` (output = flattened block content text; `isError` = block `is_error`; `detail` = envelope `toolUseResult`) |
| `type:"assistant"`, content array | per block: `text`→`assistant_message` (model from `message.model`), `thinking`→`thinking`, `tool_use`→`tool_call` (toolName=name, toolUseId=id, input) |
| `type:"summary"` | `summary` (kind `title`) — also feeds session title |
| `type:"system"` | `error` if subtype indicates error, else `raw` |
| `attachment`, `queue-operation`, `last-prompt`, `file-history-snapshot`, `progress`, unknown | `raw` |

- `isSidechain: true` envelope → `sidechain: true` payload flag on all events
  from that line.
- Timestamps: envelope `timestamp` → event `ts`.
- Session fields: `started_at` = first line ts; `ended_at` = last line ts;
  `project_dir` = cwd; `git_branch` = gitBranch; `model` = last seen
  `message.model`; title precedence: last `summary` record > first
  non-meta `user_message` text (first 80 chars, newline-collapsed).
- Token totals: group assistant lines by `requestId`; per request take the
  LAST line's `usage`; `output_tokens` = Σ per-request output_tokens;
  `input_tokens` = the final request's `input_tokens +
  cache_read_input_tokens + cache_creation_input_tokens` (≈ final context
  size). Documented approximation; stored on the session row only (no
  `token_usage` events for Claude in v1).

### 11.3 Codex adapter (source facts: research §2)

- Roots: `~/.codex/sessions`; `filePatterns: ["**/rollout-*.jsonl"]`.
- `nativeId` = `session_meta.payload.id`, fallback filename stem.
- Line mapping (`{timestamp, type, payload}` envelope; event `ts` = envelope
  timestamp):

| Source record | Normalized events |
|---|---|
| `session_meta` | `session_meta` (cwd, cli_version→agentVersion, originator→entrypoint) |
| `turn_context` | `raw`; also update session `model` = payload.model (last wins) |
| `response_item` / `message` role=user | `user_message`; `meta:true` when text starts with `<environment_context>` or `<user_instructions>` |
| `response_item` / `message` role=assistant | `assistant_message` |
| `response_item` / `reasoning` | `thinking` (text = summary[].text joined by `\n\n`; `encrypted:true` when only `encrypted_content` present) |
| `response_item` / `function_call` | `tool_call` (toolName=name, toolUseId=call_id, input = JSON.parse(arguments) with raw-string fallback; `displayCommand` = joined command array for shell-like tools) |
| `response_item` / `local_shell_call` | `tool_call` (toolName='shell', input=action) |
| `response_item` / `function_call_output` | `tool_result` (toolUseId=call_id; output = payload.output; if output parses as JSON `{output, metadata}` use inner output and keep rest in `detail`) |
| `response_item` / `custom_tool_call`, `web_search_call`, others | `tool_call` generic (input = payload) or `raw` if no call semantics |
| `event_msg` / `token_count` | `token_usage` snapshot from `info.total_token_usage` (cumulative → session totals use LAST snapshot) |
| `event_msg` / `user_message`, `agent_message` | **skipped** (duplicates of response_item — research §2.3; validation step in issue 07) |
| `event_msg` / others (`task_started`, `task_complete`, ...) | `raw` |
| `compacted` | `summary` (kind `compaction`) |
| unknown type | `raw` |

- Title: first non-meta `user_message` (80 chars). `started_at` =
  session_meta ts; `ended_at` = last record ts.

## 12. File-change extraction (`src/ingest/filechange.ts`)

Runs inside the importer, downstream of adapters: it observes the normalized
`tool_call`/`tool_result` stream and emits derived `file_change` events
appended immediately after the triggering `tool_result` (or after the
`tool_call` if no result is seen by end of file), with the same `ts`, and
`derivedFrom = toolUseId`.

| Trigger | Extraction |
|---|---|
| Claude `Write` call | kind = `create`, or `modify` when `tool_result.detail.type == "update"`; diff = whole-content unified diff against empty (create) or omitted for modify-without-original (then `fragment:true`, diff from detail.originalFile when present) |
| Claude `Edit` call | kind `modify`; diff = `createTwoFilesPatch(path, path, old_string, new_string)` with `fragment:true`; when `detail.structuredPatch` exists, convert it to a unified diff instead (`fragment:false`) |
| Claude `MultiEdit` call | kind `modify`; concatenated fragment diffs per edit |
| Claude `NotebookEdit` call | kind `modify`; no diff, path only |
| Codex `apply_patch` function_call, or shell command containing `apply_patch` with a `*** Begin Patch` heredoc | parse envelope grammar (research §2.4) → one `file_change` per file section: Add→`create`, Update→`modify` (+`renameFrom` on Move to), Delete→`delete`; diff = section body converted to unified diff form |
| anything else (bash redirects, git commands) | NOT extracted (non-goal §3.2) |

- `additions`/`deletions` counted from diff `+`/`-` lines (excluding headers).
- Paths normalized: absolute when resolvable against session `project_dir`,
  stored as-is otherwise; `filePath` column always set.
- Parser failures degrade to no `file_change` plus one `error(origin:parser)`
  event; never abort the import.

## 13. Incremental sync engine (`src/ingest/importer.ts`)

### 13.1 Algorithm (per `blackbox sync` run)

```
acquire exclusive lock (<dataDir>/.lock, proper-lockfile; fail fast with message)
for each enabled adapter:
  for each root in config.sources[adapter].roots:
    files = fast-glob(adapter.filePatterns, {cwd: root, absolute: true})
    for each file:
      st = stat(file)
      row = source_files[path]
      if row.ignored → skip
      if row && st.size == row.size_bytes && st.mtimeMs == row.mtime_ms → skip (unchanged)
      firstLine = read first line
      if row && (st.size < row.bytes_ingested
                 || sha256(firstLine) != row.first_line_hash) → REWRITTEN:
          delete session(row.session_id) events + session row (keep blobs per §10),
          reset row offsets to 0
      nativeId = adapter.sessionKeyForFile(file, firstLine); if null → skip
      ctx = adapter.createContext(...)
      open read stream at offset row.bytes_ingested
      read COMPLETE lines only (buffer partial tail; do not consume it)
      batch: for each line → adapter.parseLine → filechange.process → normalize
      in ONE transaction per file batch:
        upsert session (id, source, native_id, imported_at)
        insert events with seq = current max(seq)+1…  (UNIQUE(session_id,seq) guards)
        write blobs + event_blobs
        merge adapter.sessionPatch(ctx) into session row
        recompute session counters (SQL COUNT by type)
        update source_files {bytes_ingested += consumed, lines_ingested, size, mtime,
                             first_line_hash (if was 0), session_id, last_synced_at,
                             error_count}
release lock; print summary table (files scanned/imported/skipped, events added,
parse errors, duration)
```

### 13.2 Properties & edge cases

- **Idempotent**: unchanged files skip on stat; offsets guarantee append-only
  files never re-import lines. `--full` flag: treat every file as REWRITTEN
  (wipe & re-import) — the recovery path for adapter bug fixes.
- **Partial last line**: never consumed; picked up next sync once the newline
  arrives (agents append whole lines, but torn reads are expected mid-write).
- **Deleted source files**: rows remain; sessions remain (recorder semantics —
  history outlives the source). `source_files.path` missing on disk → skip.
- **Concurrency**: single writer via lockfile; readers (`ui`, `search`) work
  concurrently thanks to WAL.
- **Crash safety**: offsets update in the same transaction as event inserts —
  a killed process resumes exactly where the last committed batch ended.
- **Error budget**: per-line parse errors are recorded (`error` events +
  `source_files.error_count`) and summarized; a file aborts (with error log)
  only on I/O failure, leaving other files unaffected.
- Options: `--source claude-code|codex`, `--root <dir>` (extra root, once),
  `--full`, `--json` (machine-readable summary), `--quiet`.

### 13.3 Watch mode (`blackbox watch`)

Loop: run one sync pass → sleep `sync.watchIntervalSeconds` (flag
`--interval <s>` overrides) → repeat. SIGINT/SIGTERM → finish current file
batch, release lock, exit 0, print totals. Each pass prints one status line
only when changes were found (`--quiet` suppresses; `--verbose` prints every
pass). No fs-event dependency (polling is sufficient and portable).

## 14. Session lifecycle & stats

- States are implicit: a session exists once any of its events are imported;
  "live" sessions simply keep growing on subsequent syncs. No state column.
- `ended_at` is always "last event so far" — the UI labels a session *active*
  when `ended_at` is within 2 × watch interval of now AND its source file
  mtime advanced in the latest sync (computed at query time; no stored flag).
- Stats (in `show` and `GET /api/sessions/:id`): event counts by type, top 10
  tools by call count, top 20 changed files with add/del totals, duration
  (`ended_at - started_at`), token totals, source file path, parse error count.

## 15. Search

### 15.1 Query semantics

- Input: free-text string + filters. The string is split on whitespace; each
  term is FTS5-quoted (`"term"` with internal `"` doubled) and AND-joined.
  No user-facing operators in v1.
- Terms with < 3 characters (trigram minimum, counting Unicode code points):
  the whole query falls back to `LIKE '%term%' ESCAPE '\'` scanning over
  `events.text` with all filters applied and `LIMIT` enforced (documented
  slower path; used for e.g. `rm`, 2-char Japanese words).
- Filters (all optional, AND-combined): `types[]`, `tools[]`, `sources[]`,
  `sessionRef` (prefix-resolved), `projectSubstring`, `pathSubstring`,
  `since`/`until` (ISO date or datetime, compared to event `ts`).

### 15.2 SQL shape (FTS path)

```sql
SELECT e.id, e.session_id, e.seq, e.ts, e.type, e.tool_name, e.file_path,
       snippet(events_fts, 0, char(1), char(2), '…', 24) AS snip,
       bm25(events_fts) AS rank
FROM events_fts
JOIN events e ON e.id = events_fts.rowid
JOIN sessions s ON s.id = e.session_id
WHERE events_fts MATCH :ftsQuery
  AND (…filters on e.*, s.*…)
ORDER BY rank
LIMIT :limit OFFSET :offset;
```

`char(1)`/`char(2)` markers are converted to ANSI colors (CLI) or `<mark>`
(viewer). Total count via separate `COUNT(*)` query (same WHERE).

### 15.3 Index coverage

Only the derived `text` (capped at `fts.maxIndexedTextBytes`, default 64 KiB),
`tool_name`, `file_path` are indexed. `text_truncated=1` rows are flagged in
results ("matched text may be incomplete"). Ranking: `bm25(events_fts)`
ascending, ties by `ts` descending.

## 16. CLI specification

Binary names: `blackbox` and alias `bbx`. Global flags: `--data-dir <dir>`,
`--verbose`, `--quiet`, `--no-color`, `--version`, `--help`. Exit codes:
`0` ok, `1` runtime error, `2` usage error, `3` session not found,
`4` ambiguous session prefix (candidates listed on stderr).
All list/tabular commands support `--json` (stable machine-readable shape;
one JSON document to stdout, nothing else on stdout).

### 16.1 `blackbox sync` / `blackbox watch` — §13.

### 16.2 `blackbox list`

`blackbox list [--source s] [--project substr] [--since d] [--until d]
[--limit n=50] [--json]`
Columns: `ID` (nativeId first 8), `SOURCE`, `TITLE` (≤60), `PROJECT`
(basename), `STARTED` (local time), `DUR`, `EVENTS`, `TOOLS`, `FILES`,
`TOKENS` (`in/out`). Sorted `started_at` DESC. Footer: `N of M sessions`.

### 16.3 `blackbox show <ref>`

Summary block (all session fields) + stats tables (§14). `--json` for the
same data. `--events` additionally streams the full event list in replay
format (no timing).

### 16.4 `blackbox search`

`blackbox search <query…> [--type t,…] [--tool t,…] [--source s,…]
[--session ref] [--project substr] [--path substr] [--since d] [--until d]
[--limit n=50] [--offset n] [--json]`
Table: `SESSION` (8-char id), `SEQ`, `TS` (local), `TYPE`, `TOOL`, `SNIPPET`
(ANSI-highlighted, single line, ≤120 cols). Footer: total count + hint
`blackbox replay <id> --seek <seq>`.

### 16.5 `blackbox delete <ref> [--yes] [--forget]`

Confirmation prompt unless `--yes` (or non-TTY → require `--yes`). Deletes
session + events (+FTS via trigger) + orphaned blobs; marks its
`source_files.ignored=1` so sync does not resurrect it. `--forget` instead
deletes the `source_files` row (next sync re-imports).

### 16.6 `blackbox replay` — §17.

### 16.7 `blackbox export`

`blackbox export <ref> --format bbox|html|md|json [-o <path>]
[--no-redact] [--yes] [--include-raw]`
Defaults: format `html`; output `./<source>-<nativeId8>.<ext>`
(`.bbox`/`.html`/`.md`/`.json`). Redaction ON by default (§21);
`--no-redact` prints a warning and requires `--yes` or interactive confirm.
`--include-raw`: include `raw`-type events (excluded by default from md/html
narrative formats; always included in bbox/json).
Prints: output path, size, redaction summary (rule → replacement count).

### 16.8 `blackbox import <file.bbox> [--force]`

Validates manifest `formatVersion == 1`. Session id collision → skip with
message (or replace with `--force`). Prints imported session ids.

### 16.9 `blackbox ui [--port n] [--no-open]`

Starts the HTTP server (§18) on `127.0.0.1:<port>` (default from config,
7710). Port busy → try port+1..+9, else exit 1 with message. Opens the
browser (macOS `open`, Linux `xdg-open`) unless `--no-open`. Ctrl-C stops.

## 17. Terminal replay (`src/replay/render.ts`)

`blackbox replay <ref> [--step] [--speed x=1] [--seek seq] [--types t,…]
[--no-thinking] [--no-tools] [--full]`

- **Renderer** (pure function `renderEvent(event, opts) → string[]`, reused by
  `show --events`): header line
  `[seq] HH:MM:SS <TYPE> <tool> <path>` colored per type (user=cyan,
  assistant=white, thinking=dim, tool_call=yellow, tool_result=dim yellow
  (red when isError), file_change=green/red, summary=magenta, error=red,
  raw=dim); body indented 2 spaces; body capped at 20 lines with
  `… (+N lines, blob: <hash8>)` tail unless `--full`; diffs colored by
  leading `+`/`-`.
- **Player**: iterate events in seq order from `--seek`; inter-event delay =
  `min(actual ts gap, 1500ms) / speed`; delay 0 when either ts missing.
- **Controls** (TTY, raw mode): `space` pause/resume, `→`/`enter` single step,
  `+`/`-` speed ×2/÷2 (bounds 0.25–16), `q` clean exit. `--step` starts
  paused. Non-TTY: no timing, no controls — plain sequential dump (makes
  `blackbox replay X | less` work).
- Sidechain events render with a `⑂` badge and extra indent.

## 18. HTTP API (express app factory `src/server/app.ts`)

Bind `127.0.0.1` only. JSON responses `{ok:true, ...}` or
`{ok:false, error:{code, message}}` with proper HTTP status. No auth (ADR-001).
CORS: same-origin only (no CORS headers needed; viewer is served by this
server).

| Method & path | Query/body | Response (`ok:true` +) |
|---|---|---|
| GET `/api/health` | — | `{version, dataDir, sessionCount}` |
| GET `/api/sessions` | `source,project,q(title substr),since,until,sort=started_at,order=desc,offset=0,limit=50(max 200)` | `{total, items: SessionSummary[]}` |
| GET `/api/sessions/:id` | — | `{session: SessionSummary, stats: SessionStats}` |
| GET `/api/sessions/:id/events` | `offset=0,limit=200(max 1000),types,tools` | `{total, items: BlackboxEvent[]}` (seq order; payload blob refs resolved to previews only) |
| GET `/api/sessions/:id/events/:seq` | — | `{event: BlackboxEvent}` with blob fields inlined up to 1 MiB each |
| GET `/api/search` | §15 filters + `q,offset,limit=50(max 200)` | `{total, hits: [{event, sessionSummary, snippetHtmlSafe}]}` |
| GET `/api/blobs/:hash` | — | raw `text/plain; charset=utf-8` body (404 if unknown) |

`SessionSummary` = session row fields camelCased. `:id` accepts full id or
unambiguous nativeId prefix (404 vs 409 on ambiguity). Static: serves
`dist/viewer/` at `/` with SPA fallback to `index.html` for non-`/api` paths.

## 19. Viewer (web UI)

React SPA, HashRouter (works from `file://` in embedded mode). Data access
ONLY through the `DataProvider` interface:

```ts
interface DataProvider {
  mode: 'http' | 'embedded';
  listSessions(q): Promise<{total, items}>;
  getSession(id): Promise<{session, stats}>;
  getEvents(id, {offset, limit, types}): Promise<{total, items}>;
  getEvent(id, seq): Promise<BlackboxEvent>;
  search(params): Promise<{total, hits}>;   // embedded: client-side substring scan
  blobText(hash, maxBytes?): Promise<string | null>;
}
```

Routes & pages:

| Route | Page | Key behaviors |
|---|---|---|
| `#/` | SessionList | table ≈ CLI `list` columns; filter bar (source select, project text, date range); click → timeline. Hidden in embedded mode (redirects to the single session's timeline). |
| `#/session/:id?seq=N&types=…` | Timeline | paged event cards (200/page, "Load more" top & bottom); per-type filter chips; collapse/expand card bodies (default collapsed beyond 12 lines); tool_call/tool_result visually paired by toolUseId (result nested under call); DiffView renders unified diffs with +/- coloring and add/del counts; `seq` param scrolls to & highlights that event; sidechain badge; keyboard j/k = next/prev card. |
| `#/session/:id/replay?seq=N` | Replay | timeline data + player chrome: play/pause, step fwd/back, speed (0.5/1/2/4/8), progress scrubber (by seq), current-event autoscroll; timing = capped recorded gaps as §17. |
| `#/search?q=…&filters…` | Search | form mirroring CLI filters; results list (session, ts, type, snippet with `<mark>`); click → `#/session/:id?seq=N`. |

Styling: single hand-written `viewer/src/styles.css`, dark theme, monospace
for payloads; no CSS framework. Long text always wraps inside cards
(`overflow-x: auto` for diffs); the page body never scrolls horizontally.

## 20. Export & sharing

All exports run through the redaction engine first (§21) unless `--no-redact`.

### 20.1 JSON (`--format json`)

Single JSON document: `{formatVersion:1, exportedAt, session, events:[…]}`
with blob refs inlined as full text (no cap) — the fidelity format.

### 20.2 `.bbox` bundle (`--format bbox`)

Zip (adm-zip) containing:

```
manifest.json    {formatVersion:1, tool:"blackbox", toolVersion, exportedAt,
                  sessions:[{id, eventCount}], redaction:{applied, rules:[{name,count}]}}
sessions/<id with ':'→'_'>/session.json    # session row fields
sessions/<id with ':'→'_'>/events.jsonl    # one BlackboxEvent per line ($blob refs kept)
blobs/<hash>                                # every referenced blob (post-redaction)
```

Import (§16.8) is the exact inverse; `imported_from` set on the session row;
`source` preserved. Round-trip (export → import into empty dir → export) must
be byte-identical for `events.jsonl` (test in issue 20). In-memory zip cap:
warn above 200 MiB estimated.

### 20.3 Single-file HTML (`--format html`)

- Template: `dist/viewer-embed/index.html` — the Vite single-file build
  (vite-plugin-singlefile) containing the placeholder comment
  `<!--BLACKBOX_DATA-->` before `</body>`.
- Exporter replaces the placeholder with
  `<script id="blackbox-data" type="application/json">…</script>` where the
  JSON is `{formatVersion:1, exportedAt, session, events, blobs:{hash→text},
  truncatedBlobs:[{hash, originalSize}], redaction}`.
- JSON is escaped for script context: `</` → `<\/`, U+2028/U+2029 escaped.
- Blobs larger than `blob.htmlInlineCapBytes` (256 KiB) are truncated to a
  4 KiB preview and listed in `truncatedBlobs`; the viewer shows a
  "truncated on export" notice on affected fields.
- The output must be fully self-contained: no `src=`/`href=` to external
  origins (asserted by test), works from `file://`.
- Viewer boot: if `#blackbox-data` element exists → EmbeddedProvider (single
  session; search = client-side substring over loaded events).

### 20.4 Markdown (`--format md`)

Human-readable narrative: front-matter-style header (session fields), then one
`##` section per event: `## [seq] HH:MM:SS type — tool/path`, body as
blockquote (messages) or fenced code (tool I/O, capped 100 lines with
truncation note), diffs in ```` ```diff ```` fences. `raw` events omitted
unless `--include-raw`.

## 21. Redaction engine (`src/export/redact.ts`)

- Applied at export time only; the local DB always keeps originals.
- Operates on every string field of session/events/blobs via a JSON walker
  (payloads walked recursively), replacing matches with
  `«redacted:<ruleName>»`.
- Built-in rules (name → JS regex, all with `g` flag):

| name | pattern (summary) |
|---|---|
| anthropic-key | `sk-ant-[A-Za-z0-9_-]{20,}` |
| openai-key | `sk-(proj-)?[A-Za-z0-9_-]{20,}` |
| github-token | `gh[pousr]_[A-Za-z0-9]{36,}` and `github_pat_[A-Za-z0-9_]{22,}` |
| aws-access-key | `\bAKIA[0-9A-Z]{16}\b` |
| aws-secret-heuristic | `(?i)aws.{0,20}(secret|key)['"\s:=]{1,4}[A-Za-z0-9/+=]{40}` |
| google-api-key | `\bAIza[0-9A-Za-z_-]{35}\b` |
| slack-token | `xox[baprs]-[A-Za-z0-9-]{10,}` |
| jwt | `\beyJ[A-Za-z0-9_-]{8,}\.eyJ[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\b` |
| bearer-header | `(?i)(authorization\s*:\s*bearer\s+)\S+` (keeps prefix) |
| private-key-block | `-----BEGIN [A-Z ]*PRIVATE KEY-----[\s\S]*?-----END [A-Z ]*PRIVATE KEY-----` |
| env-secret-assignment | `(?i)\b([A-Z0-9_]*(api_?key|secret|token|password|passwd|credential)[A-Z0-9_]*)\s*[=:]\s*['"]?[^\s'"]{8,}` (replaces the value only, keeps the name) |

- Custom rules from `config.redaction.customPatterns` (validated: name
  non-empty unique, pattern compiles; invalid → startup error).
- Output: `RedactionReport {applied: boolean, rules: [{name, count}], total}`
  printed by export and embedded in manifests/data islands.
- False positives are acceptable (over-redaction is the safe direction);
  `--no-redact` exists for trusted-destination exports.

## 22. Error handling & logging

- `src/util/log.ts`: `log.info/warn/error/debug` → stderr, `[blackbox]`
  prefix, colors unless `--no-color`/non-TTY. `debug` only with `--verbose`
  or `BLACKBOX_DEBUG=1`. stdout is reserved for command payload output
  (tables, JSON, exports to stdout are never mixed with logs).
- CLI top-level handler: known errors (typed `BlackboxError` with `exitCode`)
  → single-line message; unknown → message + stack only when `--verbose`.
- Parser/import errors: never fatal (§13.2); surfaced in sync summary and
  stored (`error` events, `source_files.error_count`).
- Server errors: 500 with `{ok:false}`; request log lines only in `--verbose`.

## 23. Performance budgets (verified in issue 23)

| Operation | Budget (dev machine, Apple Silicon) |
|---|---|
| `sync` throughput | ≥ 20 MB/min of JSONL (≈ 5k lines/s) |
| `sync` no-change pass over 1k files | < 2 s |
| `search` (FTS path) on 500k-event DB | < 500 ms for top 50 |
| Timeline API page (200 events) | < 150 ms |
| Viewer first paint of 10k-event session | < 1.5 s (paged) |
| HTML export of 5k-event session | < 15 s, output < 30 MB |
| DB size overhead | < 3× raw JSONL size (incl. FTS) |

Synthetic corpus generator (issue 05) provides the measurement fixtures.

## 24. Security & privacy

- All data local; server binds 127.0.0.1; no telemetry, no network calls at
  runtime (the only outbound action is the user explicitly sharing a file).
- Exports redact by default (§21); `--no-redact` is explicit + confirmed.
- Committed fixtures are synthetic only — never copied from real transcripts
  (issue 05 guards; CI grep for obvious real-key patterns as belt&braces).
- Data dir created 0700. `.bbox` import parses zip entries with path
  traversal guards (reject entries containing `..` or absolute paths).
- HTML export escapes all user content when rendering (React default) and the
  data island per §20.3 — no script injection from transcript content.

## 25. Testing strategy

### 25.1 Layers

| Layer | Tooling | Scope |
|---|---|---|
| Unit | vitest, colocated `*.test.ts` | model, adapters (fixture-driven), filechange, redact, search query builder, renderer (snapshot) |
| Integration | vitest + tmpdir (`fs.mkdtemp`) | importer against fixture trees, migrations, blob store, delete/orphan logic |
| CLI e2e | vitest + execa on built `dist/` | every command happy-path + key errors, `--json` shapes, exit codes |
| API | vitest + supertest (app factory, no listen) | every endpoint incl. pagination, 404/409 |
| Viewer | vitest + jsdom + @testing-library/react | DataProvider (mocked), page rendering, embedded-mode boot |
| System e2e | node script (issue 23) | generate corpus → sync → search → export html/bbox → import → assert |

### 25.2 Fixtures

`fixtures/claude-code/*.jsonl`, `fixtures/codex/*.jsonl` — synthetic sessions
generated by `scripts/make-fixtures.ts` covering: plain chat, tool calls
(Bash/Read/Edit/Write/MultiEdit; shell/apply_patch), thinking/reasoning,
sidechain lines, tool errors, unknown record types, malformed JSON line,
multibyte Japanese text, an oversized (>64 KiB) tool output, summary/compaction
records. Each fixture documents its scenario in a leading comment line using
the source's own comment-safe channel (a `raw`-mapped record), because JSONL
has no comments.
A local-only validation command (`npm run validate:local`, NOT in CI) runs
both adapters over the real `~/.claude/projects` and `~/.codex/sessions` trees
in dry-run mode and prints: files parsed, events by type, unknown-type counts,
parse errors — the acceptance gate for issues 06/07.

### 25.3 Conventions

- No network in tests; no reliance on real home-dir data in CI.
- Deterministic: fixed timestamps inside fixtures; snapshot tests strip
  volatile fields (durations, versions).
- Every issue lists its required tests in Acceptance Criteria; CI green is a
  merge precondition for every issue.

## 26. CI (GitHub Actions, issue 01)

`.github/workflows/ci.yml`: on push + PR. Matrix: `macos-14`, `ubuntu-24.04`;
Node 22 (`actions/setup-node`, npm cache). Steps: `npm ci` →
`npm run lint` → `npm run typecheck` → `npm test` → `npm run build`
(tsc + viewer + viewer-embed builds). No artifact publishing.

## 27. Versioning & compatibility

- `meta.format_version = 1`: bump only with a migration; `blackbox` refuses
  to open a NEWER format DB ("upgrade blackbox") and auto-migrates older ones.
- `.bbox` `manifest.formatVersion = 1`: importer accepts `== 1` only (clear
  error otherwise). HTML data island carries the same version.
- Package version starts `0.1.0`; releases are git tags (issue 24), no npm
  publish in v1.

## 28. Top risks

| Risk | Impact | Mitigation |
|---|---|---|
| Source format churn (both agents ship weekly) | parse gaps | tolerant parsers + `raw` fallback + fixtures + local validation gate (issues 05–07); `sync --full` recovery path |
| FTS trigram index bloat on huge diffs | disk, slow writes | per-event text cap; budget check (issue 23) |
| Huge sessions (100k+ events) in viewer | UX freeze | server-side pagination everywhere; no full-session loads in http mode |
| Codex canonical-stream choice wrong for some session kinds | missing/duplicate messages | issue 07 validation step on real data before finalizing |
| Redaction misses a secret family | leak on share | over-broad defaults, custom patterns, default-on, explicit `--no-redact` gate |
| better-sqlite3 native build failures | install breakage | CI on both OSes from issue 01; engines pin |

## 29. Known unknowns (may spawn new issues during implementation)

1. Exact `toolUseResult` shapes per Claude tool version (`structuredPatch`
   availability) — resolve during issue 06/08 via local validation.
2. Codex `event_msg` vs `response_item` coverage on non-interactive
   (`codex exec`) sessions — resolve during issue 07.
3. Whether Claude sidechain/sub-agent transcripts appear as separate files in
   current versions — resolve during issue 05/06; may add a linking field
   (v1 fallback: they import as independent sessions).
4. Real-world FTS index size ratio — measured in issue 23; may force a lower
   default text cap.
5. `snippet()` behavior with trigram tokenizer on CJK — verify in issue 12;
   fallback: hand-rolled snippet around first match offset.
6. adm-zip memory ceiling for very large bundles — measured in issue 23;
   fallback: switch to yazl streaming (ADR-003 amendment).

## 30. Glossary

| Term | Meaning |
|---|---|
| source | An agent product whose transcripts we ingest (`claude-code`, `codex`) |
| session | One recorded agent run (= one transcript file, normalized) |
| event | One normalized timeline entry (§8.1 types) |
| replay | Ordered re-presentation of events with recorded timing; never re-execution |
| blob | Content-addressed file holding an oversized payload field |
| `.bbox` | blackbox's portable zip bundle format (§20.2) |
| redaction | Pattern-based secret masking applied to exports (§21) |
| sidechain | Claude Code sub-agent content embedded in a session |
