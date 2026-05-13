# Station Pre-loading Design

**Date:** 2026-05-13
**Status:** Approved
**Feature:** Load the next station's audio stream in the background while the current one is playing, so forward navigation is instant.

---

## Problem

When the user swipes/presses next, `loadAndPlay()` creates a new `AudioPlayer`, calls `player.play(url)`, and waits for the stream to connect and buffer (typically 500ms–3s). During this wait the "connecting…" overlay is visible. Pre-loading eliminates this wait for fast-skipping users.

---

## Architecture

Three targeted changes:

1. **New `usePreload` hook** (`src/usePreload.ts`) — owns all pre-loading state and lifecycle.
2. **`AudioPlayer.setVolume(v: number)`** — one-line addition so the pre-loaded player can buffer silently at volume 0.
3. **App.tsx wiring** — calls `prime` after each station starts, `consume` at the top of the forward-navigation path, and `reprimeIfNeeded` on pointer activity.

---

## `usePreload` Hook

### Signature

```typescript
function usePreload(
  catalog: Station[] | null,
  savedStations: Station[],
  selectNext: (catalog: Station[], saved: Station[], last: Station | null) => Station,
): {
  prime: (currentStation: Station) => void
  consume: () => { player: AudioPlayer; station: Station } | null
  reprimeIfNeeded: (currentStation: Station) => void
}
```

### Internal state

```typescript
const preloadRef = useRef<{
  player: AudioPlayer
  station: Station
  ready: boolean
  expiryTimer: ReturnType<typeof setTimeout>
} | null>(null)
```

### `prime(currentStation)`

1. Stops any existing pre-load player and cancels its expiry timer (`preloadRef.current?.player.stop()`).
2. Clears `preloadRef.current = null`.
3. If `catalog` is null, returns early.
4. Calls `selectNext(catalog, savedStations, currentStation)` to pick the next station. This updates `recentlyPlayed` in the taste profile (same side-effect as normal navigation).
5. Creates a new `AudioPlayer` and sets volume to 0 (`player.setVolume(0)`).
6. Calls `player.play(nextStation.url, false)` as fire-and-forget:
   - On resolve → sets `preloadRef.current.ready = true`.
   - On reject → stops player and sets `preloadRef.current = null` (silent failure).
7. Sets `preloadRef.current = { player, station: nextStation, ready: false, expiryTimer }`.
8. Starts a 30-second expiry timer. When it fires → stops the player and sets `preloadRef.current = null`.

### `consume()`

- If `preloadRef.current?.ready` is true: clears ref (including cancelling expiry timer), returns `{ player, station }`.
- Otherwise returns `null`. Callers fall back to normal loading — no regression.

### `reprimeIfNeeded(currentStation)`

- If `preloadRef.current` is null: calls `prime(currentStation)`.
- Otherwise does nothing (a no-op when a pre-load is already active).
- Safe to call on every pointer/mouse event (just a ref check when active).

### Cleanup

On unmount: stops pre-loaded player and cancels expiry timer.

---

## `AudioPlayer` Change

Add one method:

```typescript
setVolume(v: number): void {
  this.audio.volume = v
}
```

---

## App.tsx Integration

### After a station starts playing

At the end of `loadAndPlay` (both initial tap path and navigation path), just before `return true`:

```typescript
prime(s)
```

This replaces any existing pre-load with a fresh one for the station that just started.

### Forward-navigation path in `loadAndPlay`

```typescript
const preloaded = !stationToLoad ? consume() : null
const s = stationToLoad ?? preloaded?.station ?? pickNext(catalog, savedStations, stationRef.current)
const player = preloaded?.player ?? new AudioPlayer()
if (!preloaded) await player.play(s.url, false)   // skipped if pre-loaded
oldPlayer?.fadeOut(400)   // crossfade starts immediately when pre-loaded
playerRef.current = player
player.fadeIn(1000)
// ...
prime(s)  // queue next pre-load
```

When `stationToLoad` is provided (`goPrevious`), `preloaded` is null. `prime(prev)` at the end stops any stale pre-load and queues a fresh one from the station we just went back to.

### Pointer activity (re-prime after expiry)

On the overlay div (which only captures events when `phase === 'playing'`):

```tsx
onPointerMove={() => station && reprimeIfNeeded(station)}
onPointerDown={e => { station && reprimeIfNeeded(station); handlePointerDown(e) }}
```

`onPointerMove` covers mouse movement and touch drag. `onPointerDown` covers tap-without-move. Both together ensure the first sign of user intent re-starts the pre-load if it has expired.

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Pre-load `play()` fails | Silent: ref cleared, `consume()` returns null, normal loading proceeds |
| `consume()` returns null (not ready, or expired) | Normal loading proceeds — no regression |
| User navigates before pre-load is ready | `consume()` returns null, normal loading |
| User navigates before 30s expiry | `consume()` returns ready player, instant crossfade |
| `goPrevious` ignores pre-load | Pre-load stopped by subsequent `prime()` call |

---

## Resource Management

- At most one background stream open at a time.
- Pre-load is always stopped before a new one starts (`prime` cancels previous).
- 30-second expiry drops the stream if the user settles in.
- Activity (pointer move/down) re-primes on demand after expiry.
- Unmount cleanup stops any active pre-load.

---

## Testing

- `usePreload.test.ts` — unit tests with mocked `AudioPlayer`:
  - `prime` starts a pre-load and marks it ready on resolve
  - `prime` replaces an existing pre-load (stops old player)
  - `consume` returns ready pre-load and clears ref
  - `consume` returns null when not ready
  - 30s expiry stops player and clears ref
  - `reprimeIfNeeded` is a no-op when pre-load is active
  - `reprimeIfNeeded` calls prime when ref is null
  - Unmount cleanup stops active pre-load
- `audioPlayer.test.ts` — add test for `setVolume`
- App.tsx integration: existing navigation tests continue to pass (consume returning null = current behaviour)
