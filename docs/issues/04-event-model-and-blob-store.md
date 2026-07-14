# Title

Normalized event model, text derivation, payload offload, blob store

## Summary

Implement the core domain layer: TypeScript types for the normalized event
model (DESIGN §8), the content-addressed blob store (DESIGN §10), and
`normalize.ts` which converts adapter event drafts into DB-ready rows —
deriving searchable text, capping it, and offloading oversized payload strings
into blobs.

## Context

Adapters (06/07) emit `NormalizedEventDraft`s; the importer (09) persists
them. This issue provides the pure middle layer both sides depend on, so it
must land before any adapter work and be exhaustively unit-tested.

## Scope

- `src/model/events.ts`, `src/model/normalize.ts`, `src/blob/store.ts`
- colocated tests

## Detailed Requirements

1. `src/model/events.ts`:
   - `EventType` union exactly per DESIGN §8.1; payload interfaces per the
     table (SessionMetaPayload, UserMessagePayload, …, RawPayload), each with
     optional `sidechain?: boolean`.
   - `NormalizedEventDraft = { ts: string | null; type: EventType;
     role: Role | null; toolName?: string; toolUseId?: string;
     filePath?: string; payload: <per-type> }`.
   - `BlackboxEvent` envelope per DESIGN §8.2.
   - Type guards `isToolCall(e)`, `isFileChange(e)`, etc. where useful.
   - `SessionFields` interface matching the sessions table columns
     (camelCase).
2. `src/blob/store.ts`:
   - `class BlobStore { constructor(blobDir: string, db: Db) }`
   - `put(text: string) → {hash, size}`: sha256 hex of UTF-8 bytes; path
     `<blobDir>/<hash[0..2]>/<hash>`; if file exists → return (dedupe);
     else write to `<path>.tmp-<pid>` then `fs.renameSync` (atomic);
     upsert `blobs` row (INSERT OR IGNORE).
   - `read(hash, opts?: {maxBytes?: number}) → string | null` (null when
     missing; truncate at maxBytes on read when provided).
   - `deleteOrphans(db) → number`: delete `blobs` rows (and files) whose hash
     is absent from `event_blobs`; returns count. Used by session delete
     (issue 11) — implemented and tested here.
3. `src/model/normalize.ts`:
   - `deriveText(draft) → string` per DESIGN §8.1 table:
     message/thinking/summary/error → their text; tool_call → `toolName + ' '
     + (displayCommand ?? JSON.stringify(input))`; tool_result → output;
     file_change → `path + '\n' + (diff ?? '')`; raw →
     `JSON.stringify(json)` capped 4096 bytes; session_meta/token_usage → ''.
   - `finalizeEvent(draft, {ftsCapBytes, offloadThresholdBytes, blobStore})
     → {row: EventRowInsert, blobRefs: {hash, size, field}[]}`:
     1. text = deriveText, UTF-8-truncate at ftsCapBytes on a char boundary,
        set `text_truncated` when cut.
     2. Walk `payload` recursively (objects/arrays); every **string** leaf
        longer than offloadThresholdBytes (UTF-8 bytes) → `blobStore.put`,
        replace with `{"$blob": {hash, size, preview: first 1024 chars}}`,
        record `{field: '<dot.path>', hash, size}`.
     3. Serialize payload JSON.
   - UTF-8 truncation helper must never split a multibyte char (test with
     Japanese).
   - `resolveBlobRefs(payloadJson, blobStore, {maxBytesPerField}) → payload`
     — inverse used by API/exports (inline up to cap, else keep ref).
4. Determinism: identical input drafts produce byte-identical rows and blob
   hashes (no timestamps generated here; `imported_at` is the importer's job).

## Acceptance Criteria

- [ ] Unit tests: deriveText for every event type; cap + `text_truncated`
      flag; multibyte-safe truncation (no U+FFFD in output); offload replaces
      only oversized strings and preserves structure; nested arrays/objects
      walked; preview is 1024 chars; blob dedupe (two puts, one file);
      atomic-write leftover `.tmp-` files are cleaned or absent after put;
      `deleteOrphans` removes exactly unreferenced blobs (file + row);
      `resolveBlobRefs` round-trips content under cap and keeps `$blob` above
      cap.
- [ ] Property-ish test: normalize(draft) is deterministic across two runs.
- [ ] No DB writes besides `blobs` upsert inside BlobStore (importer owns the
      rest) — enforced by module boundaries (no imports from ingest/).
- [ ] Lint/typecheck/CI green.

## Validation

Run vitest with coverage locally: `model/` and `blob/` files ≥ 90% line
coverage (repo does not enforce a global gate; this is this issue's local
bar, checked in review).

## Dependencies

- 03 (Db, blobs table)

## Non-goals

- No adapter logic, no file reading, no session-row writes.
- No binary blob support (v1 blobs are UTF-8 text; DESIGN §20.3 note).

## Design References

- DESIGN §8 (model), §10 (blob store), §15.3 (text cap), §20.3 (inline caps)
