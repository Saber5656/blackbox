# Title

Project scaffold: TypeScript ESM package, toolchain, CLI skeleton, CI

## Summary

Create the npm package skeleton for blackbox exactly as specified in DESIGN §6
and ADR-003: strict-TypeScript ESM, commander-based CLI entry with global
flags, logging utility, lint/format/test toolchain, and a GitHub Actions CI
matrix (macOS + Linux). After this issue, `blackbox --version` and
`blackbox --help` work from a global install and CI is green.

## Context

The repository currently contains only README.md and docs/. Every later issue
assumes this exact layout, toolchain, and CI. Getting pins and module settings
right here prevents ESM/CJS breakage later (better-sqlite3 is CJS-native,
imported from ESM).

## Scope

- `package.json`, `tsconfig.json`, `vitest.config.ts`, `eslint.config.js`,
  `.prettierrc.json`, `.gitignore`, `.github/workflows/ci.yml`
- `src/cli/index.ts` (program shell), `src/util/log.ts`, `src/util/table.ts`,
  `src/util/paths.ts` (tilde expansion), placeholder command registration
- npm scripts: `build`, `build:viewer` (stub, no-op until issue 15), `dev`,
  `lint`, `typecheck`, `test`, `validate:local` (stub printing "not yet
  implemented", exit 1)

## Detailed Requirements

1. `package.json`:
   - `"name": "blackbox"`, `"version": "0.1.0"`, `"private": true`,
     `"type": "module"`, `"engines": {"node": ">=20"}`
   - `"bin": {"blackbox": "dist/cli/index.js", "bbx": "dist/cli/index.js"}`
   - Dependencies (exact majors per ADR-003): `commander@^12`,
     `picocolors@^1`. Dev: `typescript@^5`, `vitest@^2`, `tsx`, `eslint@^9`,
     `typescript-eslint`, `prettier@^3`, `@types/node@^22`, `execa@^9`.
     Do NOT add other deps yet (later issues add their own).
2. `tsconfig.json`: `"strict": true`, `"module": "NodeNext"`,
   `"moduleResolution": "NodeNext"`, `"target": "ES2022"`,
   `"outDir": "dist"`, `"rootDir": "src"`, `"declaration": false`,
   `"sourceMap": true`; include `src`, exclude `viewer`, `dist`, `fixtures`.
3. `src/cli/index.ts`:
   - shebang `#!/usr/bin/env node`; commander program `blackbox` with
     `.version()` read from package.json (via `createRequire` or fs read —
     no JSON import assertions).
   - Global options: `--data-dir <dir>`, `--verbose`, `--quiet`,
     `--no-color` (DESIGN §16). Store on a `GlobalOpts` object passed to
     commands.
   - Register placeholder subcommands `sync watch list show delete search
     replay export import ui` that print
     `error: '<name>' is not implemented yet` to stderr and exit 1. Each later
     issue replaces its placeholder.
   - Top-level error handler per DESIGN §22: `BlackboxError` class
     (`message`, `exitCode`) in `src/util/log.ts` or `src/util/errors.ts`;
     unknown errors print stack only with `--verbose`. Exit codes per §16.
4. `src/util/log.ts`: `info/warn/error/debug` to **stderr** with `[blackbox]`
   prefix; colors via picocolors, disabled when `--no-color` or
   `!process.stderr.isTTY`; `debug` gated on `--verbose` or
   `BLACKBOX_DEBUG=1`.
5. `src/util/table.ts`: `renderTable(headers: string[], rows: string[][],
   opts?: {maxWidths?: number[]}) → string` — left-aligned, two-space gaps,
   cell truncation with `…` at maxWidth. Unit-tested.
6. `src/util/paths.ts`: `expandTilde(p)`, `defaultDataDir()`
   (`~/.blackbox`).
7. Lint: eslint 9 flat config with typescript-eslint recommended +
   prettier compat; `npm run lint` covers `src` (viewer added later).
8. CI `.github/workflows/ci.yml` per DESIGN §26: push+PR triggers, matrix
   `os: [macos-14, ubuntu-24.04]`, Node 22 with npm cache; steps
   `npm ci`, `npm run lint`, `npm run typecheck`, `npm test`, `npm run build`.
9. `.gitignore`: `dist/`, `node_modules/`, `*.tsbuildinfo`, `.DS_Store`.

## Acceptance Criteria

- [ ] `npm ci && npm run build` succeeds on a clean checkout (Node 20 and 22).
- [ ] `node dist/cli/index.js --version` prints `0.1.0`; `--help` lists all
      ten subcommands; unknown subcommand exits 2 (commander default
      configured accordingly).
- [ ] `node dist/cli/index.js sync` exits 1 with the not-implemented message
      on stderr and nothing on stdout.
- [ ] `npm install -g .` (or `npm link`) exposes both `blackbox` and `bbx`.
- [ ] Unit tests exist and pass for `table.ts` (truncation, alignment) and
      `log.ts` (debug gating, color off when no-color).
- [ ] CI workflow runs green on both matrix OSes.
- [ ] `npm run lint` and `npm run typecheck` pass with zero warnings.

## Validation

1. Fresh clone → `npm ci && npm run build && node dist/cli/index.js --help`.
2. `npx vitest run` locally; confirm the same commands CI runs.
3. Open a draft PR and confirm both matrix jobs pass before merge.

## Dependencies

None (first issue).

## Non-goals

- No database, config, adapters, server, or viewer code.
- No additional dependencies beyond the list above.
- No npm publish configuration.

## Design References

- DESIGN §4 (architecture), §6 (layout), §16 (CLI global flags/exit codes),
  §22 (logging), §26 (CI)
- ADR-003 (stack pins)
