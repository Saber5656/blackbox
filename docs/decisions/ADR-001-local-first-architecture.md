# ADR-001: Local-first CLI + embedded SQLite, no server service in v1

Date: 2026-07-05
Status: Accepted

## Context

blackbox records, replays, searches, and shares agent runs (Claude Code, Codex
CLI). Candidate architectures:

- **A. Local-first CLI**: single binary/package, SQLite index on the user's
  machine, local web viewer, file-based sharing.
- **B. Client + self-hosted server**: daemon with HTTP API and web UI,
  multi-machine.
- **C. Hosted SaaS**: accounts, upload, team features.

Transcripts contain prompts, code, file contents, and potentially secrets.
The primary v1 user is an individual running agents on one machine.

## Decision

Choose **A. Local-first CLI**.

- All data stays in `~/.blackbox` (SQLite + content-addressed blob store).
- The web viewer is served by `blackbox ui` bound to `127.0.0.1` only.
- Sharing is explicit file export (`.bbox` bundle, single-file HTML) with
  redaction applied by default at export time.

## Consequences

- No auth, no TLS, no multi-user concerns in v1 → drastically smaller surface.
- Sharing is asynchronous (send a file), not a live URL. Acceptable for v1.
- A future server/team mode (v2+) can reuse the same normalized event model and
  export bundle format as its wire format.
- Search scale is bounded by one machine's SQLite; budgets sized accordingly
  (DESIGN.md §23).
