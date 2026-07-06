# Title

Single-file HTML export (self-contained shareable viewer)

## Summary

Implement `blackbox export --format html` per DESIGN §20.3: take the
`dist/viewer-embed/index.html` single-file build, inject the redacted
session as a JSON data island replacing the `<!--BLACKBOX_DATA-->`
placeholder with correct script-context escaping, enforce blob inline caps
with truncation notices, and guarantee the output works offline from
`file://` with zero external requests. This is the US-5 deliverable.

## Context

The embedded viewer machinery exists (issue 15: EmbeddedProvider, embed
build, placeholder; 16/17/18: pages). The exporter composes them with the
redaction engine (19) through the common export pipeline (20).

## Scope

- `src/export/html.ts` (replaces the stub registered in issue 20)
- template-location logic + build assertion
- tests (node-side; no browser automation)

## Detailed Requirements

1. Template resolution: `dist/viewer-embed/index.html` located relative to
   the compiled module (`new URL('../../viewer-embed/index.html',
   import.meta.url)` — verify the actual relative path from
   `dist/export/html.js`). Missing template → `BlackboxError`
   ("run npm run build:viewer"). Template must contain the placeholder
   exactly once — zero or multiple → error (guards against build drift).
2. Data island construction (DESIGN §20.3):
   - Shape: `{formatVersion: 1, exportedAt, session, events, blobs:
     {hash→text}, truncatedBlobs: [{hash, originalSize}], redaction}`.
   - Blob cap: blobs with redacted size > `blob.htmlInlineCapBytes`
     (default 262144) → include only first 4096 chars in `blobs` AND list
     in `truncatedBlobs` (the EmbeddedProvider from issue 15 already
     surfaces `truncated: true`).
   - Serialization safety: after `JSON.stringify`, apply replacements
     `</` → `<\/`, ` ` → `\\u2028`, ` ` → `\\u2029`
     (script-context escaping; test with payload containing literal
     `</script>` and U+2028).
   - Injected element: `<script id="blackbox-data"
     type="application/json">…</script>` replacing the placeholder comment.
3. Self-containment assertions (in the exporter as a post-write check AND
   as tests): output contains no `src="http`, `href="http`, `url(http`,
   `@import`, `fetch(` outside the bundled app code — implement as: scan
   for `https?://` occurrences and allow only those inside the data island
   (user content may contain URLs); mechanically: assert no `https?://`
   BEFORE the data island script tag. Fail export on violation.
4. Size handling: final size logged; > 50 MiB → warning suggesting
   `--format bbox`; no hard limit.
5. `--include-raw` (DESIGN §16.7): default html export FILTERS OUT
   `raw`-type events from `events` (narrative format); flag includes them.
   Event seq values are preserved as-is (gaps allowed; EmbeddedProvider
   slices by array index — verify it uses seq for display but index for
   paging; adjust provider if it assumed dense seq — test with a gapped
   fixture).
6. Determinism: same inputs + fixed exportedAt (inject via opts for tests)
   → byte-identical output.

## Acceptance Criteria

- [ ] Test: export fixture session → output parses as HTML (string checks
      sufficient): contains exactly one `id="blackbox-data"` script; JSON
      island parses (`JSON.parse` after extracting between the script
      tags); `</script>`-containing payload does not terminate the island
      (extract-and-parse proves it); U+2028 case parses.
- [ ] Blob cap test: >256 KiB fixture blob → 4096-char preview +
      truncatedBlobs entry; under-cap blob inlined fully.
- [ ] Self-containment: no `https?://` before the data island; template
      placeholder count guards (0 and 2 cases error).
- [ ] Redaction: exported island contains zero built-in-rule matches
      (fixture with fake secrets).
- [ ] raw events excluded by default, included with `--include-raw`;
      gapped-seq paging works in EmbeddedProvider (jsdom test added on the
      viewer side).
- [ ] Manual gate (checklist in PR): open the exported file of a REAL
      session via `file://` in Chrome and Safari with network disabled
      (devtools offline): session list redirects to timeline; timeline
      renders diffs; replay plays; embedded search hits; truncated-blob
      notice visible; devtools network tab shows zero external requests.
- [ ] Determinism test (fixed exportedAt) byte-identical across two runs.
- [ ] Lint/typecheck/CI green.

## Validation

Local: export a real session (`blackbox export <ref>` — html is the
default), send the file to another machine or browser profile, open
offline, walk the manual gate checklist. Attach checklist + file size to PR.

## Dependencies

- 16, 17 (embedded pages), 19 (redaction), 20 (export pipeline + stub swap)
- 18 recommended merged (embedded search) — wave order guarantees it

## Non-goals

- No PDF/gif exports; no multi-session HTML; no server-hosted share links
  (v2); no minification beyond what the vite embed build already does.

## Design References

- DESIGN §16.7, §19 (EmbeddedProvider), §20.3, §21, §23 (size budget),
  §24 (escaping)
