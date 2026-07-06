# Title

Viewer scaffold: Vite React app, DataProvider abstraction, session list page

## Summary

Create the `viewer/` Vite + React 18 + TypeScript app with HashRouter, the
`DataProvider` interface with its `HttpProvider` and `EmbeddedProvider`
implementations (embedded = data island, DESIGN §20.3), the dark-theme CSS
baseline, the build integration (`dist/viewer` + `dist/viewer-embed`
single-file build with the data placeholder), and the first page: SessionList,
per DESIGN §19.

## Context

Every viewer page (16–18) and the HTML exporter (21) depend on the provider
abstraction and the two build outputs defined here. The embedded build must
contain the literal placeholder `<!--BLACKBOX_DATA-->` that issue 21 replaces.

## Scope

- deps (viewer package.json — the viewer is an npm workspace? NO: keep ONE
  root package.json; add viewer deps there): `react@^18`, `react-dom@^18`,
  `react-router-dom@^6`, dev: `vite@^5`, `@vitejs/plugin-react`,
  `vite-plugin-singlefile@^2`, `jsdom`, `@testing-library/react`,
  `@testing-library/user-event`
- `viewer/index.html`, `viewer/vite.config.ts`, `viewer/vite.embed.config.ts`
- `viewer/src/{main.tsx, App.tsx, styles.css}`
- `viewer/src/data/provider.ts` (+ http.ts, embedded.ts, types.ts mirroring
  DESIGN §18 JSON contracts)
- `viewer/src/pages/SessionList.tsx` + components (FilterBar, SessionTable)
- npm scripts: `build:viewer` (both configs), root `build` runs tsc + both
  viewer builds; `dev:viewer` (vite dev with proxy `/api` →
  `http://127.0.0.1:7710`)
- vitest jsdom environment for `viewer/**/*.test.tsx`

## Detailed Requirements

1. Build outputs:
   - `vite.config.ts`: `base: './'`, outDir `../dist/viewer` (emptyOutDir),
     standard chunks.
   - `vite.embed.config.ts`: vite-plugin-singlefile (all JS/CSS inlined),
     outDir `../dist/viewer-embed`; `viewer/index.html` contains
     `<!--BLACKBOX_DATA-->` immediately before `</body>` (present in BOTH
     builds; harmless in http mode).
   - No external URLs anywhere (fonts/CDN forbidden; system font stack).
2. Boot logic (`main.tsx`): if
   `document.getElementById('blackbox-data')` exists → parse JSON →
   `EmbeddedProvider(data)`; else `HttpProvider(fetch, baseUrl='')`.
   Provider exposed via React context `DataProviderContext`.
3. `DataProvider` interface exactly per DESIGN §19 (six methods + `mode`).
   - HttpProvider: thin fetch wrappers; non-`ok:true` envelope → typed
     `ApiError {code, message}` thrown; all query params serialized
     identically to DESIGN §18 names.
   - EmbeddedProvider: constructed from the §20.3 data island shape
     `{session, events, blobs, truncatedBlobs, redaction}`;
     `listSessions` → single-item result; `getEvents` slices the in-memory
     array honoring offset/limit/types; `getEvent` inlines from `blobs`
     map; `search(params)` → case-insensitive substring scan over
     `event.text`-equivalent (embedded events carry `text`), returns hits
     shaped like the API; `blobText` from the map, `truncatedBlobs` →
     returns preview + `truncated:true` metadata (extend return type:
     `{text, truncated}`).
4. Routing (`App.tsx`): HashRouter with routes exactly per DESIGN §19 table;
   unknown route → redirect `#/`. Embedded mode: `#/` redirects to
   `#/session/<id>`; a top notice bar shows
   `exported <exportedAt> · redaction <applied|off>`.
5. SessionList page:
   - On mount + on filter change: `listSessions({source, project, q, since,
     until, offset, limit:50})`; state in URL query params (shareable).
   - FilterBar: source `<select>` (all/claude-code/codex), project text
     input (debounced 300 ms), title search input `q`, date `since`/`until`
     inputs (type=date), reset button.
   - SessionTable columns = CLI list columns (ID8, source badge, title,
     project basename, started local, duration, events, tools, files,
     tokens); row click → `#/session/<id>`; header shows `N of M`;
     `Load more` button appends the next page (offset += 50).
   - Empty state: "no sessions — run `blackbox sync`".
   - `active` sessions (from `GET /api/sessions/:id`… not available in list;
     omit here) — NOT in list v1; do not call per-row detail.
6. Styling `styles.css`: CSS variables (dark background #111418 range,
   monospace font stack for data), badge colors per source, table styles,
   focus-visible outlines; body `overflow-x: hidden`, inner scrollers only
   (DESIGN §19).
7. Tests (jsdom): provider unit tests (HttpProvider param serialization with
   a mocked fetch; EmbeddedProvider slicing/filter/search/truncated-blob);
   SessionList renders rows from a mocked provider, filter updates URL,
   load-more appends.
8. Build smoke test (node, in vitest): after `npm run build:viewer`, assert
   `dist/viewer/index.html` exists, `dist/viewer-embed/index.html` is a
   single file containing `<!--BLACKBOX_DATA-->` and no
   `src="http`/`href="http` substrings.

## Acceptance Criteria

- [ ] `npm run build` produces working `dist/viewer` (served by issue 14's
      static path — manual check with `blackbox ui`) and
      `dist/viewer-embed/index.html` single file with placeholder.
- [ ] `blackbox ui` now serves the session list at `/` listing real
      sessions (manual, local).
- [ ] All jsdom tests + build smoke pass in CI on both OSes.
- [ ] `dev:viewer` proxy works against a running `blackbox ui --no-open`
      (documented in viewer/README-dev note within the code comment or
      docs/DESIGN.md untouched).
- [ ] No external network references in either build output (test above).
- [ ] Lint (extend eslint config to `viewer/`), typecheck (separate
      `tsconfig.viewer.json` with `jsx: react-jsx`, DOM lib), CI green.

## Validation

Local: `blackbox ui`, filter by source, click through to a (placeholder)
session route — 16 not yet merged so the route may render "timeline not
implemented" stub; the stub must exist so navigation works.

## Dependencies

- 14 (API + static serving)

## Non-goals

- No timeline/replay/search pages (16–18) beyond route stubs.
- No workspaces/monorepo split; single root package.json stands.
- No theming toggle, no i18n (UI chrome is English; data is user content).

## Design References

- DESIGN §6 (viewer layout), §18 (contracts), §19 (provider, routes,
  styling), §20.3 (embedded island + placeholder)
- ADR-003 (viewer stack)
