# Title

Viewer search page: cross-session search UI with hit navigation

## Summary

Implement `#/search` per DESIGN §19: a search form mirroring the CLI filters,
results grouped for scanning, `<mark>` snippet highlighting, pagination, and
click-through into the timeline at the hit's seq (`#/session/:id?seq=N`).
In embedded mode the same page searches within the single exported session
client-side.

## Context

This is US-2 in the browser, backed by `GET /api/search` (issue 14). The
EmbeddedProvider already implements a client-side `search` (issue 15); this
page must work identically against both providers.

## Scope

- `viewer/src/pages/Search.tsx`
- `viewer/src/components/{SearchForm,HitList}.tsx`
- tests (jsdom)

## Detailed Requirements

1. SearchForm fields → API params: free-text `q` (required, min 1 char);
   type multi-select chips (the 11 EventType values); tool text input;
   source select; project substring; path substring; since/until dates;
   session ref text input (optional). Submit on Enter or button;
   all state in URL query params (shareable/back-button-safe).
2. Results:
   - Call `provider.search(params)` with `limit: 50`, `offset` from URL.
   - HitList rows: session badge + title (linked to timeline), seq, local
     ts, type, tool, snippet rendered from the API's marked string — the
     string arrives HTML-escaped with literal `<mark>`/`</mark>` tags
     (DESIGN §18); render via a tiny parser that splits on the mark tags
     and emits `<mark>` React elements — NEVER
     `dangerouslySetInnerHTML` (XSS test required).
   - Row click → `#/session/<id>?seq=<seq>`.
   - Pagination: `Prev / Next` + `total` display; `hits N–M of T`.
   - Empty result → "no hits" with the executed filter summary.
   - Short-query note: when the provider/API used the LIKE fallback there is
     no marker difference — nothing to do; but sub-3-char queries must not
     error (test).
3. Embedded mode: session filter hidden (single session); everything else
   identical; a note `searching within this exported session` shows.
4. Loading/error states: spinner row; ApiError shows `error.message` inline
   with retry button.
5. Header nav: add a persistent top nav (Sessions | Search) to `App.tsx`
   (visible in http mode; embedded shows Timeline | Replay | Search for the
   session).

## Acceptance Criteria

- [ ] jsdom tests: form → URL params round-trip; provider called with
      exactly the DESIGN §18 param names; results render marks as `<mark>`
      elements; XSS probe snippet
      (`&lt;script&gt;<mark>hit</mark>` payload) renders text not elements;
      pagination offsets; empty state; error+retry; embedded-mode variant
      hides session field and searches client-side (mock EmbeddedProvider).
- [ ] Click-through URL carries id + seq (assert href).
- [ ] Nav links present in both modes.
- [ ] Manual (local, real data): Japanese query returns hits and highlights;
      type filter narrows; click-through lands scrolled on the right event
      (checklist in PR).
- [ ] Lint/typecheck/CI green.

## Validation

Local: search a real term (e.g. a filename you know was edited), click a
`file_change` hit, confirm the timeline opens centered and highlighted on
that diff.

## Dependencies

- 16 (timeline deep-link target), 14 (search endpoint; already merged via
  wave order)

## Non-goals

- No saved searches, no search history, no regex/operators (v2), no
  cross-session search in embedded mode (single session by definition).

## Design References

- DESIGN §15 (semantics), §18 (`/api/search` contract), §19 (Search row,
  provider search in embedded mode)
