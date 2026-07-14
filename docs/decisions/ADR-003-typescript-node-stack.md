# ADR-003: TypeScript/Node.js stack and pinned core libraries

Date: 2026-07-05
Status: Accepted

## Context

Implementation will be executed issue-by-issue by lower-capability coding
agents. The stack must be: (1) extremely well represented in training data so
agents make few idiomatic mistakes, (2) cross-platform on macOS/Linux, (3) able
to ship a CLI + local HTTP server + web viewer from one repository.

Candidates: TypeScript/Node, Go (single static binary), Rust, Python.

Go/Rust produce nicer distributables but the viewer still needs a JS toolchain,
splitting the stack. Python packaging for end-user CLIs is weak. A single
TypeScript codebase covers CLI, server, and viewer with one language.

## Decision

TypeScript everywhere, single npm package (`"private": true` in v1, no npm
publish).

| Concern | Choice | Version pin (major) |
|---|---|---|
| Runtime | Node.js | >= 20 (engines), CI on 22 |
| Module system | ESM (`"type": "module"`) | — |
| Language | TypeScript, `strict: true` | 5.x |
| CLI framework | commander | 12.x |
| SQLite driver | better-sqlite3 (sync API, bundled SQLite with FTS5) | 11.x |
| Globbing | fast-glob | 3.x |
| Diff generation | diff (jsdiff) | 5.x |
| Zip (bbox bundles) | adm-zip | 0.5.x |
| HTTP server | express | 4.x |
| Viewer | React 18 + react-router-dom 6 (HashRouter) + Vite 5 | — |
| Single-file HTML build | vite-plugin-singlefile | 2.x |
| Terminal color | picocolors | 1.x |
| Process lock | proper-lockfile | 4.x |
| Tests | vitest 2.x, @testing-library/react, supertest, execa | — |
| Lint/format | eslint 9 (flat config) + typescript-eslint + prettier 3 | — |

Rationale for the contentious picks:

- **better-sqlite3** over node:sqlite / sql.js: synchronous API (simpler code
  for implementation agents), mature prebuilds, bundled recent SQLite with
  FTS5 including the `trigram` tokenizer (needed by ADR-004).
- **express 4** over hono/fastify: maximally familiar to implementation agents;
  performance is irrelevant at localhost scale.
- **adm-zip** over archiver/yazl: simplest synchronous API. Memory cost of
  in-memory zipping is accepted for v1 (bundle size cap documented in
  DESIGN.md §20.2).
- **React 18** over Preact/Svelte: familiarity beats bundle size for a local
  tool.

## Consequences

- `npm link` (or `npm install -g .`) is the v1 install path; publishing is a
  release-phase decision (issue 24).
- better-sqlite3 is a native module; CI must run `npm ci` on both macOS and
  Linux to catch prebuild issues (issue 01).
- All dependency additions beyond this table require a design-doc update.
