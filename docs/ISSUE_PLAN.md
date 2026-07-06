# blackbox — v1 Issue Plan

Derived from docs/DESIGN.md (canonical). GitHub Issues are generated from
docs/issues/*.md; if they diverge, these files win.

## 1. v1 completion statement

v1 is complete when **all 24 issues below are merged and validated**, meaning:
a user can run `npm install -g .` on macOS or Linux and then

1. `blackbox sync` imports their complete Claude Code and Codex CLI history,
2. `blackbox watch` keeps recording while agents run,
3. `blackbox list / show / search / replay` answer user stories US-1..US-4
   (DESIGN §1.2) from the terminal,
4. `blackbox ui` serves the local viewer with session list, timeline, replay,
   and search pages,
5. `blackbox export` produces redacted `.bbox` / single-file HTML / Markdown /
   JSON artifacts and `blackbox import` round-trips `.bbox` bundles (US-5,
   US-6),
6. CI is green on macOS + Linux across lint, typecheck, unit/integration/CLI
   tests, and builds, and the performance budgets of DESIGN §23 are met,

with no product behavior defined only in prose outside these issues, except
items explicitly listed as v2 deferred (DESIGN §3.3) or known unknowns (§8
below).

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|------|-------|------|
| 01 | issues/01-project-scaffold-and-ci.md | Project scaffold, toolchain, CI pipeline | A |
| 02 | issues/02-config-and-data-directory.md | Config loading and data directory | A |
| 03 | issues/03-sqlite-storage-and-migrations.md | SQLite storage, schema v1, migration runner | A |
| 04 | issues/04-event-model-and-blob-store.md | Normalized event model, text derivation, blob store | A |
| 05 | issues/05-synthetic-fixtures-and-local-validation.md | Synthetic fixtures + local format validation harness | A |
| 06 | issues/06-claude-code-adapter.md | Claude Code transcript adapter | B |
| 07 | issues/07-codex-adapter.md | Codex rollout adapter | B |
| 08 | issues/08-file-change-extraction.md | File-change extraction (diffs) | B |
| 09 | issues/09-incremental-sync-engine.md | Incremental sync engine + `sync` command | B |
| 10 | issues/10-watch-mode.md | Watch mode (`blackbox watch`) | B |
| 11 | issues/11-session-cli-list-show-delete.md | Session CLI: list / show / delete | C |
| 12 | issues/12-search-engine-and-cli.md | Search engine + `search` command | C |
| 13 | issues/13-terminal-replay.md | Terminal replay | C |
| 14 | issues/14-http-api-server.md | HTTP API server + `ui` command | D |
| 15 | issues/15-viewer-scaffold-and-session-list.md | Viewer scaffold, DataProvider, session list | D |
| 16 | issues/16-viewer-timeline.md | Viewer timeline page | D |
| 17 | issues/17-viewer-replay-mode.md | Viewer replay mode | D |
| 18 | issues/18-viewer-search.md | Viewer search page | D |
| 19 | issues/19-redaction-engine.md | Redaction engine | E |
| 20 | issues/20-bbox-export-import.md | `.bbox` export + import | E |
| 21 | issues/21-html-export.md | Single-file HTML export | E |
| 22 | issues/22-markdown-and-json-export.md | Markdown + JSON export | E |
| 23 | issues/23-e2e-and-performance.md | System e2e + performance budget verification | F |
| 24 | issues/24-docs-and-release-prep.md | User docs, README, release prep | F |

## 3. Dependency table

`A → B` means B requires A merged first. "soft" = B can start, one section
blocks.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01, 02 |
| 04 | 03 |
| 05 | 01 |
| 06 | 04, 05 |
| 07 | 04, 05 |
| 08 | 06, 07 |
| 09 | 03, 04, 06, 07, 08 |
| 10 | 09 |
| 11 | 09 |
| 12 | 09, 11 (soft: `resolveSessionRef` for the `--session` flag only) |
| 13 | 09, 11 (session resolver) |
| 14 | 11, 12 |
| 15 | 14 |
| 16 | 15 |
| 17 | 16 |
| 18 | 16 (soft: 14 search endpoint already merged) |
| 19 | 02, 04 |
| 20 | 11, 19 |
| 21 | 16, 17, 19 |
| 22 | 11, 19 |
| 23 | 10, 12, 13, 14, 20, 21, 22 |
| 24 | 23 |

Parallelization within waves: 06 ∥ 07; 11 ∥ 12 (12's `--session` flag last,
after 11's resolver lands); 17 ∥ 18; 19 can start any time after wave A;
20 ∥ 21 ∥ 22.

```
01 ─► 02 ─► 03 ─► 04 ─┬─► 06 ─┐
 └──────► 05 ─────────┴─► 07 ─┴─► 08 ─► 09 ─► 10
                                          │
                       ┌── 11 ◄───────────┤
                       │    └─► 13        │
                       │   12 ◄───────────┘
                       ▼    │
                      14 ◄──┘
                       └─► 15 ─► 16 ─► 17 ─► 21 ◄─ 19
                                  └──► 18          │
                              20 ◄─────────────────┤
                              22 ◄─────────────────┘
                     23 ◄─ (10,12,13,14,20,21,22) ─► 24
```

## 4. Implementation waves

| Wave | Issues | Gate to close the wave |
|---|---|---|
| A — Foundation | 01–05 | CI green; `blackbox --version` runs; DB opens with schema v1; fixtures committed; `npm run validate:local` harness runs (adapters pending) |
| B — Ingestion | 06–10 | `blackbox sync` imports both fixture trees AND real local data (validate:local reports 0 crashes, unknown-type rate visible); `watch` records a live session |
| C — Query/CLI | 11–13 | US-1..US-3 satisfiable from the terminal against real data |
| D — Viewer | 14–18 | `blackbox ui` manual checklist (issue 14/18 validation sections) passes |
| E — Sharing | 19–22 | `.bbox` round-trip byte-stable; HTML export opens from `file://` offline with zero external requests; redaction report correct on fixture secrets |
| F — Hardening | 23–24 | DESIGN §23 budgets measured & met; docs complete; tag `v0.1.0` prepared (tagging itself needs explicit user authorization) |

## 5. Coverage table (DESIGN.md section → issues)

| DESIGN § | Topic | Covered by |
|---|---|---|
| 1–3 | Goals, personas, scope | all (context); scope guard in every issue's Non-goals |
| 4 | Architecture | 01 (layout), 09 (pipeline) |
| 5 | Stack (ADR-003) | 01 |
| 6 | Repository layout | 01 |
| 7 | Config & data dir | 02 |
| 8 | Event model | 04 |
| 9 | SQLite schema | 03 |
| 10 | Blob store | 04 |
| 11.1 | Adapter interface | 05 (contract tests), 06, 07 |
| 11.2 | Claude Code adapter | 06 |
| 11.3 | Codex adapter | 07 |
| 12 | File-change extraction | 08 |
| 13.1–13.2 | Sync engine | 09 |
| 13.3 | Watch mode | 10 |
| 14 | Session lifecycle & stats | 11 (stats), 14 (active flag) |
| 15 | Search | 12 (engine+CLI), 14 (API), 18 (UI) |
| 16.1 | sync/watch CLI | 09, 10 |
| 16.2–16.5 | list/show/search/delete CLI | 11, 12 |
| 16.6 | replay CLI | 13 |
| 16.7–16.8 | export/import CLI | 20, 21, 22 |
| 16.9 | ui CLI | 14 |
| 17 | Terminal replay | 13 |
| 18 | HTTP API | 14 |
| 19 | Viewer | 15, 16, 17, 18 |
| 20.1 | JSON export | 22 |
| 20.2 | .bbox | 20 |
| 20.3 | HTML export | 21 |
| 20.4 | Markdown export | 22 |
| 21 | Redaction | 19 |
| 22 | Errors & logging | 01 (log util), enforced per issue |
| 23 | Performance budgets | 23 |
| 24 | Security & privacy | 05 (fixture policy), 14 (bind/limits), 19, 20 (zip guards), 21 (escaping), 23 (checks) |
| 25 | Testing strategy | every issue; 05 (fixtures), 23 (system e2e) |
| 26 | CI | 01 |
| 27 | Versioning | 03 (format_version), 20 (bbox version), 24 (release) |
| 28–29 | Risks & unknowns | 05, 06, 07, 12, 23 (validation steps) |
| 30 | Glossary | — (reference) |

Every DESIGN section with product behavior maps to ≥ 1 issue. ✔

## 6. Validation strategy (whole product)

1. **Per-issue**: each issue's Acceptance Criteria include its tests; CI
   (lint + typecheck + vitest + build, macOS/Linux) must be green before
   merge. No issue is "done" with skipped tests.
2. **Wave gates**: table in §4 — each wave ends with a concrete demonstrable
   outcome, several against *real local data* via the `validate:local`
   harness (never in CI, never committed).
3. **Adapter truth check**: issues 06/07 cannot close until the local
   validation run reports zero parser crashes and an unknown-record-type rate
   that is enumerated (each unknown type listed → either mapped or explicitly
   `raw`-accepted).
4. **Round-trip invariants**: issue 20 (bbox export→import→export byte-equal
   `events.jsonl`), issue 21 (HTML self-containment: zero external URLs,
   parses offline).
5. **Security checks**: fixture secret corpus fully redacted (issue 19);
   zip path-traversal tests (issue 20); CI grep guard against real-looking
   keys in fixtures (issue 05).
6. **System e2e + budgets**: issue 23 runs the full chain
   (generate → sync → search → replay render → export both → import) and
   measures DESIGN §23 budgets; regressions block release.
7. **Release gate**: issue 24 manual QA checklist on a clean machine account
   (fresh `~/.blackbox`), then tag preparation. Tagging/publishing requires
   explicit user authorization (repo rule).

## 7. Deferred v2 items

See DESIGN §3.3 — live hook capture; additional agent sources; annotations;
team server; cost analytics; semantic/morphological search; fork-and-rerun;
ink TUI; blob GC; encryption at rest; Playwright e2e; npm publish; Windows CI.
None of these may be partially implemented "while nearby" (scope guard).

## 8. Known unknowns that may create additional issues

Tracked in DESIGN §29; the trigger points are:

| Unknown | Surfaces in | Likely new issue if confirmed |
|---|---|---|
| `toolUseResult`/`structuredPatch` variance | 06, 08 | extend filechange for new result shapes |
| Codex `event_msg`-only sessions (`codex exec`) | 07 | fallback stream selection logic |
| Sidechain transcripts as separate files | 05, 06 | session-linking field + UI grouping |
| FTS trigram index ratio too high | 23 | text-cap tuning / `detail=none` migration |
| `snippet()` misbehaves on CJK | 12 | hand-rolled snippet function |
| adm-zip memory ceiling | 23 | switch to streaming zip (ADR-003 amendment) |

Any confirmed unknown gets a new numbered issue file here before
implementation (docs-first rule).
