# Audio Fades — Design Spec

**Created:** 2026-05-12
**Status:** Approved
**Feature area:** Audio / AudioPlayer

---

## Goal

Eliminate the hard audio cut on station start and station skip. Replace with:
- **Fade-in** over ~1s when a station starts playing
- **Crossfade** on navigation: old stream fades out over 0.4s while new stream fades in over 1s simultaneously

---

## Approach

`HTMLAudioElement.volume` animated via `requestAnimationFrame`. No changes to the audio graph. Works regardless of whether `AudioContext` is present (i.e., works with the current `corsFriendly: false` forced state).

`requestAnimationFrame` at 60fps over 400ms = 24 frames — more than sufficient for a smooth perceptible ramp.

---

## Changes

### `AudioPlayer` (`src/audioPlayer.ts`)

Three new methods:

**`fadeIn(durationMs: number): void`**
- Sets `audio.volume = 0` immediately
- Cancels any in-progress fade
- Starts a `requestAnimationFrame` loop that steps volume linearly from 0 → 1 over `durationMs`
- Non-blocking (fire-and-forget)

**`fadeOut(durationMs: number): Promise<void>`**
- Records the start volume at call time
- Cancels any in-progress fade
- Starts a `requestAnimationFrame` loop that steps volume linearly from current → 0 over `durationMs`
- Resolves the promise when complete, then calls `stop()` internally
- Caller does not need to call `stop()` separately

**`cancelFade(): void`** (private)
- Cancels the pending `requestAnimationFrame` handle if one exists
- Called internally at the start of `fadeIn` and `fadeOut` to prevent two fades racing

`stop()` is updated to also cancel any in-progress fade and reset `audio.volume = 1` so the next `play()` starts at full volume.

### `App.tsx` (`src/App.tsx`)

**Initial load branch** (`isInitial = true`):
- After `await player.play(...)`, call `player.fadeIn(1000)`
- No await — fade runs in background while phase transitions to `'exiting'` → `'playing'`

**Navigation branch** (`isInitial = false`):
- Before loading the new stream, call `oldPlayer.fadeOut(400)` — non-blocking, old player will auto-stop after 400ms
- After `await newPlayer.play(...)`, call `newPlayer.fadeIn(1000)`
- The old `playerRef.current?.stop()` call is removed from this branch — `fadeOut` handles the stop

---

## Crossfade Timing

```
t=0ms   User hits Next → old player starts fading out (0→vol over 400ms)
        New stream begins loading in background
t=Xms   New stream ready → new player.play() at volume=0, fadeIn(1000) starts
t=400ms Old player reaches vol=0, stops
t=X+1000ms New player reaches vol=1
```

The old and new streams overlap for up to 400ms. This is intentional — it's the crossfade.

If the new stream loads in under 400ms (fast network), both fades run simultaneously and there is no silence. If the new stream takes longer than 400ms, there will be a brief silence between old stopping and new being audible — acceptable and rare.

---

## Testing

Unit tests in `src/audioPlayer.test.ts`:
- `fadeIn` sets volume to 0 immediately, then ramps to 1 after the full duration
- `fadeOut` ramps volume to 0 and calls `stop()` after the duration
- A second `fadeIn` while one is in progress cancels the first
- `stop()` cancels an in-progress fade and resets volume

Tests use fake `requestAnimationFrame` (vitest's jsdom provides it; manually step frames in tests).

---

## Out of scope

- Fade on pause/resume — not part of the spec; plain pause/play feel intentional
- Volume control / user-adjustable volume — deferred
- Fades when audio-reactive mode is re-enabled — the GainNode path will need its own fade strategy at that point; this spec covers only the current `corsFriendly: false` path
