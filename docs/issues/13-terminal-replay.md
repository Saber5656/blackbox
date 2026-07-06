# Title

Terminal replay: event renderer and interactive player

## Summary

Implement `src/replay/render.ts` (pure per-event terminal renderer shared
with `show --events`) and the `blackbox replay` command with recorded-timing
playback, keyboard controls, filters, and a non-TTY sequential dump, per
DESIGN §17. This completes US-3 in the terminal.

## Context

Replay is re-presentation, never re-execution (DESIGN §30). The renderer is
deliberately a pure function so it is snapshot-testable and reusable by
issue 11's `show --events` (which this issue now activates) and by the
markdown exporter's mental model (issue 22 has its own renderer but follows
the same section logic).

## Scope

- `src/replay/render.ts` + snapshot/unit tests
- `src/cli/commands/replay.ts` (replaces placeholder)
- Activate `blackbox show --events` (swap the issue-11 stub for the real
  renderer)

## Detailed Requirements

1. `renderEvent(event: BlackboxEvent, opts: {color: boolean, maxBodyLines:
   number | null, width: number}) → string[]` (lines, no trailing \n):
   - Header: `[<seq>] <HH:MM:SS|--:--:--> <TYPE padded 12> <tool ?> <path ?>`
     colored per the DESIGN §17 palette (picocolors; `color:false` → plain).
   - Body per type: messages/thinking → text lines; tool_call →
     `displayCommand ?? pretty-printed input JSON (2-space, sorted keys)`;
     tool_result → output (prefix `!` line `exit/error` when isError);
     file_change → `kind path (+a -d)` line then diff lines colored by
     leading char (`+` green, `-` red, `@@` cyan); token_usage → one
     `in/out` line; summary/error → text; raw → `sourceType` +
     first 3 lines of JSON; session_meta → `cwd`, `agent version` lines.
   - Body indent 2 spaces; cap at `maxBodyLines` (default 20) with tail
     `  … (+N more lines${blobHint})` where blobHint = `, full: blob
     <hash8>` when a `$blob` ref was truncated upstream.
   - `sidechain` payload flag → header prefixed `⑂ ` and body indent 4.
   - Never wraps manually; long lines pass through (terminal wraps).
2. Player `playSession(events: Iterable, opts)` in the same module:
   - Delay before event i+1 = `min(ts[i+1]-ts[i], 1500ms) / speed`; missing
     either ts → 0. Speed bounds 0.25–16.
   - TTY controls (raw mode via `process.stdin.setRawMode`): space =
     pause/resume, `→` or enter = step (works while paused), `+`/`-` =
     speed ×2/÷2 (announce `speed: 2x` on a status line), `q` / Ctrl-C =
     restore terminal + exit 0. Status line rendered with `\r` padding, not
     a TUI framework.
   - Non-TTY stdout: ignore timing and controls entirely; dump all events
     sequentially (makes `blackbox replay X | less -R` work); `--no-color`
     honored automatically via issue-01 log/color rules (respect
     `process.stdout.isTTY`).
3. `blackbox replay <ref>` flags (DESIGN §17): `--step` (start paused),
   `--speed <x>` (default 1; validate bounds), `--seek <seq>` (start at seq,
   validate 0..max), `--types t1,t2`, `--no-thinking` (drop thinking),
   `--no-tools` (drop tool_call+tool_result+file_change), `--full`
   (maxBodyLines null).
   - Events stream from the DB in seq order via a paged reader (500/batch;
     no full-session array in memory).
   - Header before playback: one session summary line
     (`<source>:<nativeId8> "<title>" <events> events <duration>`).
4. `show --events`: replaces its issue-11 stub with `renderEvent` over all
   events, no timing, no controls (plain dump path).

## Acceptance Criteria

- [ ] Renderer snapshot tests: one snapshot per event type (color OFF for
      stability), plus colored smoke asserting ANSI codes present for
      user/assistant/file_change; body-cap tail format; sidechain prefix;
      isError marker.
- [ ] Player unit tests with injected fake clock/sleep: gap capping at
      1500 ms, speed division, missing-ts → 0 delay.
- [ ] CLI integration (execa, non-TTY): `replay <fixture-session>` dumps all
      events in seq order; `--seek N` starts at N; `--types tool_call`
      filters; `--no-thinking` drops thinking; exit 0.
- [ ] `--speed 100` → usage error exit 2 (bounds).
- [ ] `show --events` now renders (cross-issue stub replaced; test updated).
- [ ] Interactive path manually verified on macOS Terminal (pause/step/
      speed/quit) — checklist in PR with a short recording or transcript.
- [ ] Lint/typecheck/CI green.

## Validation

Local: `blackbox replay <real session> --step`, walk 20 events with enter,
`q` exits cleanly, terminal state restored (no raw-mode residue: typing
works). Attach checklist to PR.

## Dependencies

- 09 (events in DB), 11 (resolveSessionRef, show stub swap)

## Non-goals

- No TUI framework/alternate screen (v2 ink browser).
- No re-execution of tools; no editing.
- No web replay (issue 17).

## Design References

- DESIGN §16.6, §17, §30 (replay definition)
