# Title

Viewer replay mode: player chrome over the timeline

## Summary

Implement `#/session/:id/replay` per DESIGN §19: the timeline data presented
one-event-at-a-time with play/pause, step forward/back, speed control,
a seq scrubber, autoscroll, and capped recorded-gap timing identical to the
terminal player (DESIGN §17).

## Context

Replay in the browser is the demo/sharing centerpiece (US-3/US-5): the HTML
export (21) ships this page for recipients. It reuses Timeline's EventCard
and DiffView; only the playback state machine is new.

## Scope

- `viewer/src/pages/Replay.tsx`
- `viewer/src/replay/player.ts` (pure playback state machine, testable
  without React)
- component/unit tests

## Detailed Requirements

1. `player.ts` — pure state machine:
   - State: `{position: number (seq index in the played sequence), playing:
     boolean, speed: 0.5|1|2|4|8, done: boolean}`.
   - Transitions: `PLAY`, `PAUSE`, `STEP_FWD`, `STEP_BACK`, `SEEK(i)`,
     `SET_SPEED(s)`, `TICK` (advance when playing).
   - `delayAfter(events, i, speed) = min(gap(ts[i], ts[i+1]), 1500) / speed`
     with missing-ts → 0 (identical constants to DESIGN §17 — import both
     players' cap from one shared constant if code sharing is impractical,
     duplicate WITH a cross-referencing comment and a test pinning 1500).
   - The machine never touches timers; the React page owns `setTimeout`
     scheduling from `delayAfter` (makes unit tests synchronous).
2. Replay page:
   - Data: fetch pages ahead of the cursor (prefetch next window when
     position crosses 75% of loaded events); never require full session
     upfront in http mode. Embedded mode has everything already.
   - Rendering: events up to `position` are shown (played history),
     current event highlighted and autoscrolled (`scrollIntoView` center);
     future events hidden. Reuses EventCard (collapsed by default except
     the current event which renders expanded).
   - Controls bar (sticky bottom): play/pause button (space), step back/fwd
     (`←`/`→`), speed cycle button (0.5→1→2→4→8, key `s`), scrubber
     `<input type=range>` over 0..eventCount-1 with seq label, elapsed
     `<current>/<total>` counter, exit link back to
     `#/session/:id?seq=<current>`.
   - `?seq=N` param: initial position (default 0); position changes update
     the URL (replaceState, no history spam).
   - Reaching the end: `done` state, play button becomes replay-from-start.
   - Page hidden (`visibilitychange`): auto-pause (avoid runaway timers).
3. Filters: replay plays ALL types v1 except a single `skip thinking`
   checkbox (persisted in URL `nothink=1`) — matches CLI `--no-thinking`.
4. A11y: controls are real buttons with aria-labels; keyboard shortcuts
   ignored when focus is in the scrubber (native arrows there).

## Acceptance Criteria

- [ ] player.ts unit tests: every transition, delay math (cap, speed,
      missing ts), done handling, seek bounds clamping — fully synchronous,
      no timers.
- [ ] Page tests (jsdom, fake timers): play advances through mocked events
      with computed delays; pause freezes; step back reveals history
      position; scrubber seek jumps; prefetch triggers at 75% (mock call
      audit); visibilitychange pauses; end→done→restart.
- [ ] URL seq sync (replaceState spy) and exit link carries current seq.
- [ ] Keyboard space/arrows/s work; ignored while range input focused.
- [ ] Manual: replay a real 500+ event session at 4x; smooth autoscroll; no
      memory blowup (windowed history is acceptable if needed — but v1
      keeps played cards mounted up to 2 000, beyond that unmount from top
      with a "… earlier events collapsed" marker; test this threshold).
- [ ] Lint/typecheck/CI green.

## Validation

Local checklist in PR: play/pause/step/speed/scrub/seek-link on a real
session; embedded-mode replay verified after issue 21 lands (cross-check
noted there).

## Dependencies

- 16 (EventCard/DiffView, timeline data plumbing)

## Non-goals

- No audio/gif export of replays; no re-execution; no live-follow of an
  active session (v2 tail mode).

## Design References

- DESIGN §17 (timing constants), §19 (Replay row), §30 (replay definition)
