# ADR-002: Ingest existing transcript files via adapters; no live hook instrumentation in v1

Date: 2026-07-05
Status: Accepted

## Context

To record agent runs we can either:

- **A. Ingest transcript files** that Claude Code and Codex CLI already write
  (`~/.claude/projects/**/*.jsonl`, `~/.codex/sessions/**/rollout-*.jsonl`),
  parsing them into a normalized event model.
- **B. Live instrumentation**: Claude Code hooks, wrapper processes, OTel
  exporters, or PTY recording around each agent invocation.

Option B captures data the files may lack (exact wall-clock of rendering,
stdout of the host terminal) but requires per-agent integration, configuration
changes on the user's machine, and breaks whenever agents change hook APIs. It
also cannot backfill history.

Option A backfills months of existing history immediately, requires zero
configuration of the recorded agents, and gets "live enough" recording via
incremental re-scanning (watch mode) because both agents append to their
transcript files during a run.

## Decision

Choose **A. File ingestion via source adapters**, with a polling **watch mode**
(`blackbox watch`, default 5s interval) reusing the same incremental import
engine for near-real-time capture. Live hook capture is deferred to v2
(DESIGN.md §3.3).

## Consequences

- Fidelity is bounded by what the agents persist. Some data (e.g. per-tool
  wall-clock duration in Claude Code) is approximated or absent; the event
  model marks such fields optional.
- Format churn is the top product risk → adapters must be fixture-tested and
  tolerant (unknown record → `raw` event, never a crash). See
  docs/research/transcript-formats.md §4.
- Watch mode is simple (stat + offset resume, no fs-event dependency) and safe
  to run continuously.
- Nothing prevents adding hook-based sources later: they would emit the same
  normalized events through the same importer API.
