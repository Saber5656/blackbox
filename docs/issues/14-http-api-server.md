# Title

HTTP API server and `blackbox ui` command

## Summary

Implement the express app factory (`src/server/app.ts`, `src/server/routes.ts`)
exposing the read-only JSON API of DESIGN §18 over the query layers from
issues 11/12, plus the `blackbox ui` command (port selection, 127.0.0.1 bind,
browser open, static viewer serving with SPA fallback) per DESIGN §16.9.

## Context

The viewer (issues 15–18) consumes exactly these endpoints through its
HttpProvider; their JSON contracts are frozen here. The server is read-only
(sync/watch write via the CLI); WAL mode makes concurrent reads safe
(DESIGN §13.2).

## Scope

- deps: `express@^4`, `@types/express`, `supertest`, `@types/supertest`
- `src/server/app.ts` (factory, no listen), `src/server/routes.ts`
- `src/cli/commands/ui.ts` (replaces placeholder)
- supertest suite

## Detailed Requirements

1. `createApp({db, blobStore, config, viewerDir?: string}) → express.App`:
   - JSON envelope: success `{ok:true, …}`; errors
     `{ok:false, error:{code, message}}` with status: 400 invalid params,
     404 not found, 409 ambiguous ref, 500 unexpected (message withheld,
     logged server-side).
   - No CORS headers (same-origin only), no auth, `x-powered-by` disabled.
   - Request log lines only in verbose mode (DESIGN §22).
2. Endpoints exactly per DESIGN §18 table:
   - `GET /api/health` → `{ok, version, dataDir, sessionCount}`.
   - `GET /api/sessions` → maps to `listSessions`; query params validated:
     `limit` int 1..200 default 50, `offset` ≥ 0, `sort` ∈ {started_at},
     `order` ∈ {asc, desc}, `q` = title substring filter (add
     `titleSubstring` to listSessions filter in this issue), `source`,
     `project`, `since`, `until` (ISO date/datetime; invalid → 400 naming
     the param).
   - `GET /api/sessions/:id` → `resolveSessionRef` semantics: not found →
     404, ambiguous → 409 with candidate list in `error.message`;
     response `{ok, session, stats}` (SessionDetail split).
     Additionally computes `active: boolean` per DESIGN §14 (ended_at within
     `2 × sync.watchIntervalSeconds` of now).
   - `GET /api/sessions/:id/events` → seq-ordered window; params `offset`
     (by seq position, default 0), `limit` 1..1000 default 200, `types`,
     `tools` (comma lists, validated against known EventType values → 400 on
     unknown). Payload blob refs stay as `$blob` refs with previews
     (DESIGN §18) — no inlining on list.
   - `GET /api/sessions/:id/events/:seq` → single event, blob refs inlined
     up to 1 MiB per field via `resolveBlobRefs`; `:seq` non-int → 400;
     missing → 404.
   - `GET /api/search` → same params as CLI search (§16.4 names:
     `q, types, tools, sources, session, project, path, since, until,
     offset, limit≤200`); hits with `snippet` where markers char(1)/(2) are
     converted to `<mark>`/`</mark>` AFTER HTML-escaping the rest
     (order matters — test).
   - `GET /api/blobs/:hash` → validate `^[0-9a-f]{64}$` else 400; stream
     file as `text/plain; charset=utf-8` with `Content-Length`; 404 when
     unknown.
3. Static serving: when `viewerDir` provided and exists → `express.static`,
   plus GET fallback to `index.html` for non-`/api/*` paths (SPA). Until
   issue 15 lands, `blackbox ui` runs API-only and logs
   "viewer assets not built; API available at /api" (so this issue is
   testable standalone).
4. `blackbox ui` (DESIGN §16.9): flags `--port <n>`, `--no-open`.
   Bind `127.0.0.1` explicitly. Port busy (`EADDRINUSE`) → try port+1..+9
   sequentially, else exit 1 listing the tried range. On listen: print
   `blackbox ui: http://127.0.0.1:<port>` and open the browser
   (`open` on darwin, `xdg-open` on linux, spawn detached, failures only
   debug-logged) unless `--no-open`. SIGINT → close server, exit 0.
5. DB opened read-only? No — open normally (WAL) but the server executes
   only SELECTs; enforce by code review + a test that the app factory never
   receives the importer.

## Acceptance Criteria

- [ ] supertest suite covers every endpoint: happy path, each 400 validation
      (bad limit, bad type name, bad hash, bad since), 404 session, 409
      ambiguous (two fixture sessions sharing a prefix constructed in seed),
      404 event seq, blob streaming byte-equality, search `<mark>` escaping
      (payload containing `<script>` stays escaped), pagination totals.
- [ ] Endpoint JSON field names verified against DESIGN §18 literally
      (camelCase session fields; `{total, items}` shapes) in tests.
- [ ] `blackbox ui --no-open` starts, health responds, Ctrl-C exits 0
      (execa integration test with SIGINT).
- [ ] Port-conflict fallback test (occupy 7710 in-test, assert 7711 chosen
      and printed).
- [ ] Server refuses nothing on localhost but binds only 127.0.0.1
      (assert via `server.address()`).
- [ ] Lint/typecheck/CI green.

## Validation

Local: `blackbox ui --no-open` + `curl http://127.0.0.1:7710/api/health`,
`/api/sessions?limit=3`, one `/api/search?q=…` against real data; attach
outputs to PR.

## Dependencies

- 11 (sessions/stats/resolve), 12 (search)

## Non-goals

- No viewer assets (15), no write endpoints, no auth/TLS, no CORS for other
  origins, no WebSocket/live-tail (v2).

## Design References

- DESIGN §14 (active), §16.9, §18 (contracts), §22, §24 (bind)
