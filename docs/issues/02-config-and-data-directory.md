# Title

Configuration loading and data directory management

## Summary

Implement `src/config/config.ts`: resolve the data directory, create it on
first use, load and validate `config.json`, and merge CLI flags, environment
variables, file values, and defaults into a single typed `Config` object per
DESIGN §7. Every later module receives configuration only through this API.

## Context

DESIGN §7 defines a strict precedence order (flags > env > file > defaults)
and a small config schema. Centralizing it now prevents each command from
inventing its own path handling. Issue 01 provided `expandTilde` and
`defaultDataDir`.

## Scope

- `src/config/config.ts` (+ colocated tests)
- Wire into `src/cli/index.ts`: build `Config` once in a preAction hook,
  attach to command context.

## Detailed Requirements

1. Types:

```ts
interface Config {
  dataDir: string;                       // absolute
  dbPath: string;                        // <dataDir>/blackbox.db
  blobDir: string;                       // <dataDir>/blobs
  sources: Record<'claude-code' | 'codex', { enabled: boolean; roots: string[] }>;
  sync: { watchIntervalSeconds: number };
  blob: { offloadThresholdBytes: number; htmlInlineCapBytes: number };
  fts: { maxIndexedTextBytes: number };
  redaction: { enabled: boolean; customPatterns: { name: string; pattern: string; flags?: string }[] };
  ui: { port: number };
}
```

2. `loadConfig(opts: {dataDirFlag?: string, env?: NodeJS.ProcessEnv}) → Config`:
   - dataDir precedence: `--data-dir` flag > `BLACKBOX_HOME` env >
     `~/.blackbox`. Expand tilde, resolve to absolute.
   - Create `dataDir` and `blobDir` with mode `0o700` if missing
     (`fs.mkdirSync recursive`). Never chmod an existing dir.
   - If `<dataDir>/config.json` exists, parse it. JSON parse error →
     `BlackboxError` (exit 1) naming the file and position. Missing file is
     fine (defaults).
   - Defaults exactly as the DESIGN §7.2 example: claude-code root
     `~/.claude/projects`, codex root `~/.codex/sessions`, watch 5 s,
     offload 65536, htmlInlineCap 262144, maxIndexedText 65536, redaction
     enabled with empty customPatterns, port 7710.
3. Validation (hand-rolled, no schema library):
   - Type-check every known key; on mismatch throw `BlackboxError` with
     message `config.json: <keyPath> must be <expected>, got <actual>`.
   - Numeric bounds: watchIntervalSeconds ≥ 1; offloadThresholdBytes ≥ 4096;
     htmlInlineCapBytes ≥ 4096; maxIndexedTextBytes ≥ 1024; 1 ≤ port ≤ 65535.
   - `redaction.customPatterns`: each needs non-empty unique `name`;
     `new RegExp(pattern, flags)` must compile — else error naming the rule.
   - Unknown top-level or nested keys → `log.warn` once per key, not an error.
   - `sources.*.roots`: tilde-expanded, non-empty array of strings.
4. Export `describeConfig(config) → string` used by `--verbose` startup debug
   line (single line, no secrets involved).
5. No config writes in v1 (file is user-edited); document this in a module
   doc comment.

## Acceptance Criteria

- [ ] Unit tests cover: all-defaults (no file), file overriding subset of
      keys, `BLACKBOX_HOME` respected, flag beating env, tilde expansion,
      invalid type error message shape, invalid regex error, unknown-key
      warning (spy on log), bounds violations, 0o700 creation on fresh dir.
- [ ] Tests use `fs.mkdtemp` sandboxes; no test touches the real home dir.
- [ ] `blackbox --verbose <any placeholder cmd>` prints the resolved dataDir
      debug line to stderr.
- [ ] Running any command twice concurrently does not race on mkdir (EEXIST
      tolerated).
- [ ] Lint/typecheck/CI green.

## Validation

1. `BLACKBOX_HOME=$(mktemp -d)/bb node dist/cli/index.js --verbose list`
   → dir created 0700, placeholder error still shown (list unimplemented).
2. Write a config.json with `"ui": {"port": "abc"}` → clear error, exit 1.

## Dependencies

- 01 (scaffold, util/paths, log, error types)

## Non-goals

- No `config` subcommand; no config file writing/migration.
- No source scanning (issue 09) — this issue only resolves the roots.

## Design References

- DESIGN §7 (config), §16 (global flags), §22 (errors), §24 (0700 dir)
