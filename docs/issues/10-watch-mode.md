# Title

Watch mode: `blackbox watch` continuous recording loop

## Summary

Implement `blackbox watch` per DESIGN §13.3: a polling loop that runs one
sync pass, sleeps `sync.watchIntervalSeconds` (or `--interval`), and repeats
until SIGINT/SIGTERM, finishing the in-flight file batch before a clean exit
with cumulative totals.

## Context

Watch mode turns the importer into a live recorder (ADR-002): both agents
append to their transcript files during a run, and the offset-resuming
importer (issue 09) makes each poll cheap (stat-skip for unchanged files).

## Scope

- `src/cli/commands/watch.ts` (replaces placeholder)
- Small importer additions only if required for cancellation hooks
- `watch` integration test

## Detailed Requirements

1. Flags: `--interval <seconds>` (≥1; default from config), `--source`,
   `--quiet`, `--json` is NOT supported (reject with usage error, exit 2 —
   watch is interactive by nature).
2. Loop semantics:
   - Iteration = one `runSync` (same code path as issue 09, including the
     lockfile). Between iterations: `setTimeout` sleep; the sleep must be
     cancellable on signal.
   - Lock behavior differs from `sync`: if the lock is busy at iteration
     start, log a warn ("waiting for other writer…") and retry next
     interval instead of exiting.
   - Output per iteration (unless `--quiet`): only when
     `eventsAdded > 0 || filesRewritten > 0` print one line:
     `HH:MM:SS +<events> events across <files> files (<sources>)`.
     `--verbose` prints a line every pass including no-ops.
3. Signal handling: first SIGINT/SIGTERM → set cancel flag; if inside
   `runSync`, finish the CURRENT file batch, commit, release lock, then
   exit 0 printing cumulative totals (iterations, events added, runtime).
   Second SIGINT → immediate `process.exit(130)`.
   Cancellation checkpoint: between files in the importer loop — add an
   optional `shouldStop?: () => boolean` to `runSync` opts (checked after
   each file commit); this is the only importer change allowed.
4. Errors inside an iteration (I/O, DB busy) → log, keep looping (watch must
   survive transient failures); 5 consecutive fully-failed iterations →
   exit 1 with a clear message (matches the global retry cap policy).
5. Clock/scheduling: use `setTimeout`, not cron; drift is acceptable; do not
   overlap iterations (next sleep starts after the pass finishes).

## Acceptance Criteria

- [ ] Integration test (execa, tmp data dir + fixture tree): start watch
      with `--interval 1`; after first pass completes, append lines to a
      fixture file; within 3 s the new events exist in the DB; send SIGINT;
      process exits 0 and prints cumulative totals; DB offsets are
      consistent (re-running `sync` adds zero events).
- [ ] Signal-during-import test: SIGINT while a large synthetic file is
      importing → exit 0, no partial transaction (event count equals either
      pre-batch or post-batch commit boundary, never mid-batch).
- [ ] Lock-busy test: hold the lock externally, start watch → warn line,
      loop continues; release lock → next iteration imports.
- [ ] 5-consecutive-failure exit path unit-tested (inject failing runSync).
- [ ] `watch --json` rejected with exit 2.
- [ ] Lint/typecheck/CI green (tests must be timing-tolerant: poll DB with
      deadline instead of fixed sleeps where possible).

## Validation

Local: run `blackbox watch --interval 2` in one terminal, run a real short
Codex or Claude Code session in another, confirm the status line appears and
`blackbox list` (after issue 11) or a SQL count shows growth. Attach the
observed line to the PR.

## Dependencies

- 09 (runSync, lockfile, summary)

## Non-goals

- No fs-events/chokidar (polling only, DESIGN §13.3).
- No daemonization/launchd integration (user runs it in a terminal; v2).
- No UI/notifications.

## Design References

- DESIGN §13.3, §16.1, §22
- ADR-002 (watch as the live-capture mechanism)
