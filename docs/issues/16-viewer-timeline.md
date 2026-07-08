# Title

Viewer timeline page: paged event cards, pairing, diff view, deep links

## Summary

Implement the `#/session/:id` Timeline page per DESIGN §19: header with
session summary/stats, paged event cards (200/page, load-more both
directions), per-type filter chips, tool call/result pairing, collapsible
bodies, DiffView for `file_change`, `?seq=N` deep links with scroll +
highlight, and j/k keyboard navigation.

## Context

The timeline is the core reading surface (US-3 in the browser) and the base
for replay mode (17) and search hit navigation (18). It must stay smooth on
10k-event sessions via windowed pagination — no full-session fetch in http
mode (DESIGN §28 risk table).

## Scope

- `viewer/src/pages/Timeline.tsx`
- `viewer/src/components/{EventCard,DiffView,TypeFilterChips,SessionHeader,
  PayloadBlock}.tsx`
- component tests (jsdom)

## Detailed Requirements

1. Data flow: `getSession(id)` for header; `getEvents(id, {offset, limit:
   200, types})` windows keyed by seq offset. State: loaded window list
   (contiguous), `types` filter (URL param `types=a,b`), pending seek.
   Load-more buttons at top (when window start > 0) and bottom (while more).
   Filter change resets windows.
2. Deep link `?seq=N`: compute containing page (`floor(N/200)*200` offset
   applies only when no type filter; with filters, fetch pages forward from
   0 until the event with seq ≥ N appears — acceptable v1 simplification,
   documented in code comment), scroll into view (`scrollIntoView({block:
   'center'})`), apply a 2 s highlight class.
3. SessionHeader: title, source badge, nativeId8 with copy-to-clipboard
   button (full id), project dir, started/ended local, duration, token
   totals, counts by type (chips doubling as the filter — clicking a type
   chip toggles it in `types`), `active` indicator when detail.active, link
   `Replay ▶` → `#/session/:id/replay`, parse-error count when > 0.
4. EventCard:
   - Header row: seq, time (HH:MM:SS local), type label with color dot,
     tool name, file path (ellipsized middle), sidechain `⑂` badge,
     `text_truncated`/truncated-blob indicator `◐` with tooltip.
   - Body by type mirroring the terminal renderer semantics (DESIGN §17
     body table): messages → text (preserve newlines, linkify NOTHING —
     plain text only, XSS-safe by React default); thinking → dimmed style;
     tool_call → displayCommand or pretty JSON in `PayloadBlock`;
     tool_result → output in PayloadBlock with error styling when isError;
     token_usage/session_meta/raw → compact key-value lines; summary →
     highlighted quote block.
   - Collapse: bodies longer than 12 rendered lines collapse with
     `show N more lines` toggle; default collapsed; expanding a `$blob`
     preview triggers `blobText(hash)` on demand (http mode) with a loading
     state and 1 MiB cap notice.
   - Pairing: a `tool_result` whose toolUseId matches the nearest preceding
     `tool_call` in the loaded window renders nested inside that call's
     card (indented result section); orphan results render standalone.
     `file_change` with `derivedFrom` matching renders as a nested
     "changed files" section under the same call card AND as its own
     card is NOT duplicated (nested only when the call is in-window,
     standalone otherwise).
5. DiffView: parse unified diff text into hunks; render line rows with
   classes add/del/hunk-header/context; `+a −d` badge; horizontal scroll
   inside the block (`overflow-x:auto`); no syntax highlighting (v1);
   fragment diffs show a `fragment` badge (payload.fragment).
6. Keyboard: `j`/`k` move a focus ring to next/prev card (scroll into
   view), `Enter` toggles collapse on focused card, `r` navigates to
   replay at the focused seq. Ignore keys when a form control is focused.
7. Embedded mode: identical page; provider slicing handles paging;
   truncated blobs show the export notice (from `blobText` metadata).

## Acceptance Criteria

- [ ] jsdom tests: renders 200-event window then loads more (mock provider
      returning 450 events); type chip toggling refetches with `types`
      param and updates URL; `?seq=` scrolls (assert scrollIntoView called
      via spy) and highlights; pairing nests result+file_change under the
      call; orphan result standalone; collapse toggle; blob expand calls
      `blobText` once and caches; diff renders hunk classes and counts;
      XSS probe event (`<img onerror>` in text/payload) renders inert text
      (no element injection — assert innerHTML lacks `<img`).
- [ ] Keyboard j/k/Enter behavior tested via user-event.
- [ ] No fetch of more than `limit` events per interaction (mock call
      audit).
- [ ] Manual: 10k-event synthetic session (issue 05 generator scaled
      locally) scrolls without freezing; initial paint < 1.5 s
      (informal here; hard budget re-measured in 23).
- [ ] Lint/typecheck/CI green.

## Validation

Local against real data: open a heavy real session, verify pairing of Bash
calls/results, expand a big tool output (blob fetch), follow a
`?seq=` link from a hand-built URL, keyboard navigation. Checklist in PR.

## Dependencies

- 15 (scaffold, providers, styles)

## Non-goals

- No replay chrome (17), no search page (18), no annotations, no syntax
  highlighting, no virtualized list library (windowed pagination only).

## Design References

- DESIGN §17 (body semantics parity), §18 (events endpoints), §19
  (Timeline row), §28 (huge-session risk)
