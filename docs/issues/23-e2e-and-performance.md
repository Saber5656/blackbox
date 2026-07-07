# Title

System end-to-end test and performance budget verification

## Summary

Build the synthetic large-corpus generator mode, a scripted full-chain
system test (generate → sync → list/show → search → replay render → export
html+bbox+md+json → import → verify), and the measurement harness that
checks every DESIGN §23 performance budget, resolving the open scale
unknowns (§29 #4 FTS ratio, #6 adm-zip ceiling).

## Context

Waves A–E delivered features issue-by-issue; this issue proves they compose
at realistic scale and pins performance numbers before release (issue 24).
Budgets failing here → fix in this issue when localized, or file a new
docs-first issue when structural.

## Scope

- `scripts/make-fixtures.ts` extension: `--scale <sessions>x<events>` mode
  producing deterministic large corpora into a target dir (not committed)
- `scripts/e2e.ts` (node script, run via `npm run e2e`) + CI job (small
  scale) + local big-scale mode
- `scripts/bench.ts` (`npm run bench`, local-only) producing a Markdown
  report table
- `docs/research/perf-baseline.md` — committed results (numbers only)

## Detailed Requirements

1. Corpus generator: deterministic (seeded PRNG with fixed seed constant,
   no Date.now) mix per session: 60% tool call/result pairs (Bash-like with
   0.5–8 KiB outputs), 15% messages (mixed Japanese/English), 10%
   thinking, 10% file_change-producing edits (Edit/apply_patch with 20–200
   line diffs), 5% edge records; ~1% oversized (>64 KiB) outputs. Emits
   BOTH source formats (claude-code and codex trees) to exercise both
   adapters.
2. `npm run e2e` (default CI scale: 20 sessions × 500 events; big scale
   flag `--big`: 100 × 5000 ≈ 500k events):
   sequence with assertions after each step —
   1. generate corpus into tmp roots
   2. `blackbox sync` (execa, `--json`) → eventsAdded == expected exactly
   3. second `sync` → eventsAdded == 0
   4. `list --json` totals; `show --json` on a known session → counter
      cross-check vs generator manifest (the generator writes
      `manifest.json` of expected per-session counts)
   5. `search --json` for 5 planted unique markers (one per event type
      class, incl. a Japanese marker and one inside a diff) → each found
      with correct session+seq
   6. `replay` non-TTY dump of one session → line count sanity + contains
      planted marker
   7. `export` all four formats; html island parses; bbox round-trip into a
      second BLACKBOX_HOME → re-export → events.jsonl byte-equal
   8. `delete --yes` one session → `sync` → not resurrected
   9. teardown
3. `npm run bench` (local, `--big` corpus) measures against DESIGN §23:
   sync MB/min, no-change pass over ≥1k files, FTS search p50/p95 over 20
   planted queries, timeline API page latency (spin up `createApp` +
   supertest timing), html export duration+size for the 5k-event session,
   DB-size/raw-JSONL ratio. Output: markdown table with budget / measured /
   pass-fail columns.
4. Unknowns resolution (docs-first updates in this PR):
   - FTS ratio (§29 #4): record measured ratio in
     docs/research/perf-baseline.md; if DB size > 3× raw → lower
     `fts.maxIndexedTextBytes` default in DESIGN §7 + config code, note in
     ADR-004.
   - adm-zip ceiling (§29 #6): bbox-export the big session while sampling
     `process.memoryUsage().rss`; > 1.5 GiB rss → file the streaming-zip
     follow-up issue per ISSUE_PLAN §8 (do not fix inline).
5. CI wiring: e2e (small) as a separate workflow job needing build;
   runtime target < 5 min per OS. bench NOT in CI (local script; results
   committed as the baseline doc).
6. Any budget miss that is a localized fix (missing index, N+1 query,
   pragma) → fix here with a regression assertion in bench; structural
   misses → new issue file + ISSUE_PLAN update (docs-first), release
   blocked until resolved or budget consciously revised in DESIGN §23 with
   rationale.

## Acceptance Criteria

- [ ] `npm run e2e` passes on both CI OSes at small scale; all 9 steps
      assert (no warnings-only steps).
- [ ] Big-scale e2e + bench run locally; `docs/research/perf-baseline.md`
      committed with the full budget table and every DESIGN §23 row
      measured; misses handled per req 6 (fixed or filed).
- [ ] FTS ratio + adm-zip rss numbers recorded; unknowns #4/#6 marked
      resolved (or spawned issues) in ISSUE_PLAN §8 table.
- [ ] Generator determinism: two runs → identical corpora (hash check in
      test).
- [ ] Lint/typecheck/CI green including the new e2e job.

## Validation

CI e2e job green ×2 OS; reviewer re-runs `npm run bench` locally and
compares within ±30% of the committed baseline (hardware variance
tolerated; direction matters).

## Dependencies

- 10, 12, 13, 14, 20, 21, 22 (full chain)

## Non-goals

- No browser-automation e2e (Playwright is v2; html verification stays
  string/JSON-level + issue-21's manual gate).
- No load testing of the HTTP server beyond single-user latency.
- No optimization work beyond budget misses.

## Design References

- DESIGN §13.2 (idempotence), §20.2–20.3 (round-trip/island), §23
  (budgets), §25.1 (system e2e layer), §29 (#4, #6)
