# Landing Card Redesign

**Date:** 2026-05-24  
**Status:** Approved, pending implementation

---

## Problem

The current full-screen `idle` landing phase adds friction. The app feels static until the user taps — the visualization runs but no station context is shown. The full-screen overlay also obscures what makes Spectrale compelling: the live visualization.

---

## Decision

Replace the full-screen idle landing with a compact frosted-glass `LandingCard` that floats over the fully live app. The station loads immediately on mount; the card is the only audio gate.

---

## Phase model

The `idle` phase is eliminated. Phases become: `loading → exiting → playing`.

On mount the app immediately enters `loading` and starts the catalog fetch + first station pre-connection (behavior unchanged from today — this already happened during idle). Phases now describe audio state only; whether the user has seen the landing card is tracked separately via a `hasStarted` boolean ref in `App.tsx`.

---

## LandingCard component

New self-contained component. Renders a centered frosted-glass panel over the running app.

**Layout (vertical stack, centered):**
```
spectrale              ← Cormorant Garamond 300, clamp(36px, 6vw, 64px), rgba(255,255,255,0.6)

Radio is cool again    ← system-ui 300, ~16px, rgba(255,255,255,0.5), letter-spacing 0.12em

── CTA block (system-ui uppercase, 12-13px, rgba(255,255,255,0.55)) ──
tap to play            ← line 1
[Station Name]         ← line 2: truncated with ellipsis, slightly brighter rgba(255,255,255,0.75)
and 14,000+ worldwide  ← line 3
```

While catalog is loading, the CTA block collapses to a single line: `tap anywhere to start`.

`backdrop-filter: blur(16px)` degrades gracefully on Firefox (no blur flag) — the opaque background color still reads clearly.

**CTA states:**

| State | Text |
|---|---|
| Catalog loading | `tap anywhere to start` |
| Station ready | `tap to play` / `[Station Name]` (truncated, 1 line) / `and 14,000+ worldwide` |

Station name: `white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 100%` — never wraps, never breaks layout.

**Card styles:**
- Background: `rgba(8, 8, 13, 0.45)`
- `backdrop-filter: blur(16px)`
- Border: `1px solid rgba(255,255,255,0.08)`
- Border radius: `16px`
- Padding: `40px 48px`
- Width: `clamp(280px, 50vw, 420px)`

**Interaction:**
- Full-screen transparent click layer (`position: fixed; inset: 0; z-index: above card`) catches tap anywhere
- `window` keydown (any non-modifier key) also triggers start — same as today
- On start: `hasStarted` → `true`, card fades out over 600ms, unmounts after fade, `loadInitial()` called

---

## Behind the card (before tap)

The full app renders immediately:
- Visualization runs (flow-field shader, high speed/drama settling over 3.5s — unchanged)
- Station info (corner or centered, per user preference) renders in skeleton state
- Skeleton resolves to real station name/country/genre/favicon once catalog loads — **before** the user taps
- Now-playing ICY metadata polling starts only after audio begins (post-tap)

The frosted card sits on top. By the time most users read the card, the real station info is already visible behind it.

---

## On tap

1. `hasStarted` → `true`
2. Card begins 600ms opacity fade-out
3. `loadInitial()` called → `play()` fires on already-buffered stream → audio starts immediately
4. Card unmounts after fade
5. Ripple rings fire on station commit (unchanged)
6. Tutorial delay starts, burst animation fires on station commit — all unchanged

No loading wait on tap. Everything is pre-rendered and pre-buffered; the tap is purely the browser audio permission gate.

---

## ShockwaveEffect removal

The `ShockwaveEffect` (center glow charge + shockwave rings) is removed from the first-load tap. It was designed for the dramatic old full-screen tap moment; with the lightweight card dismiss it is out of place.

- Pointer hold/charge handlers in `App.tsx` (`handlePointerDown`, `handlePointerUp`, `handlePointerCancel`, `chargeStartRef`) removed
- `ShockwaveEffect` class left in place (unwired) for potential future use
- Favicon no longer needs to be hidden during initial load (the "avoid conflicting with energy ball" workaround goes away)
- `RippleEffect` is unaffected — expanding rings still fire on station change and beat-driven pulses

---

## Idle-phase gate migrations

| Current gate | Replacement |
|---|---|
| `phase === 'idle'` on pointer/key handlers | `!hasStarted` |
| Top-right controls hidden during idle | Hidden while `!hasStarted` |
| `isFirstLoad && phase === 'idle'` render block | Replaced by `<LandingCard hasStarted={hasStarted} />` |
| `GenreSeed` shown during idle | Already fires 20s post-playing — no change |
| `useFirstRunTutorial` idle guard | Already triggered by `phase === 'playing'` — no change |
| `useAdaptiveQuality` FPS sampling | Already bypasses idle guard — no change |
| `isFirstLoad` flag | Kept — gates skeleton/connecting text during first load |

---

## What does not change

- Visualization start behavior (high drama, settles 3.5s)
- Skeleton → real info transition timing
- Burst animation on station commit
- Tutorial delay (2s after first audio)
- GenreSeed timing (20s after playing)
- Volume slider, controls layout, all overlays
- Deep link behavior (`tap to play [Station Name]` CTA already used this pattern)

---

## Catalog count

Use `Math.floor(catalogCounts.stations / 1000) * 1000` formatted as `"14,000+"`. Falls back to omitting the count if catalog hasn't loaded (rare — catalog and station pick arrive together).
