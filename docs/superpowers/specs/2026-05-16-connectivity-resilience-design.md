# Connectivity Resilience — Design Spec

**Date:** 2026-05-16
**Status:** Approved
**Branch:** main

---

## Problem

Four observed bugs plus two known gaps from the design spec:

**Observed bugs:**
1. "CONNECTING appears, nothing happens" — navigation has zero retries; one failed station attempt silently returns false and the user is stuck.
2. "Click next, nothing happens" — `transitioningRef` guard lives in callers (`goNext`/`goPrevious`), not in `loadAndPlay`. `SavedShelf` bypasses it entirely. Concurrent or rapid calls can wedge the guard.
3. "Back to intro screen while still playing" — if the React component unmounts and remounts (HMR, mobile browser discard/restore), all `useState` resets but the `AudioPlayer` (a ref) keeps playing. Welcome screen shows while audio is live.
4. "Two stations playing at once" — the non-preloaded navigation path calls `player.play()` (which starts at the default volume of 1), then several synchronous lines later calls `player.fadeIn()` (which sets volume to 0 and ramps up). There is a window where both old and new players are at full volume simultaneously.

**Known gaps (from design spec §17):**
5. No 4s load timeout — a stream can accept the TCP connection but never deliver audio, hanging the UI indefinitely.
6. No stall watchdog — a stream can silently drop mid-listen with no detection or recovery.
7. No auto-resume on network recovery.

---

## Approach

Consolidate all audio lifecycle logic into a new `useAudioLifecycle` hook. This fixes the root causes rather than patching symptoms:

- `transitioningRef` becomes private to the hook — no caller can bypass it.
- `loadAndPlay` disappears — replaced by typed methods (`goNext`, `goPrevious`, `loadInitial`) with shared internals.
- `usePreload` folds into the hook as an implementation detail.
- Stall watchdog, load timeout, and network recovery live alongside the retry logic they feed into.

Parameters from iOS's `StreamPlayer.swift`:
- Load timeout: **4 seconds**
- Stall timeout: **5 seconds**
- Max retries per station (goPrevious): **2**
- Max station attempts (goNext): **3 different stations**

---

## Architecture

### Responsibility split

| Concern | Owner |
|---|---|
| Player lifecycle (create, play, retry, timeout, crossfade) | `useAudioLifecycle` |
| Stall detection + watchdog | `useAudioLifecycle` |
| Network recovery | `useAudioLifecycle` |
| Preload (next station pre-connection) | `useAudioLifecycle` (folded in from `usePreload`) |
| Phase state (`idle/loading/playing`) | `useAudioLifecycle` |
| Recommendation signals | App.tsx (via `onNavigatingAway` callback) |
| Dwell timer | App.tsx (via `onStationChanged` callback) |
| Visualization wiring | App.tsx (via `onStationChanged` callback) |
| All UI rendering | App.tsx |

### Hook interface

```ts
function useAudioLifecycle(options: {
  pickNext: (catalog: Station[], saved: Station[], last: Station | null) => Station
  getSavedStations: () => Station[]
  onStationChanged: (station: Station, player: AudioPlayer) => void
  onNavigatingAway: (station: Station, listenSeconds: number) => void
}): {
  phase: Phase
  station: Station | null
  player: AudioPlayer | null       // exposed for VisualizationManager
  transitioning: boolean
  isPaused: boolean
  stallMessage: string | null
  goNext: () => void
  goPrevious: (station?: Station) => void
  loadInitial: () => void
  pause: () => void
  resume: () => void
  dismissStallMessage: () => void
  onPointerActivity: (station: Station) => void  // replaces reprimeIfNeeded
}
```

### App.tsx usage

```tsx
const {
  phase, station, player, transitioning, isPaused, stallMessage,
  goNext, goPrevious, loadInitial, pause, resume,
  dismissStallMessage, onPointerActivity,
} = useAudioLifecycle({
  pickNext,
  getSavedStations: () => savedStationsRef.current,
  onStationChanged: (s, p) => {
    vizRef.current?.loadStation({ ...s, corsFriendly: false }, p)
    vizRef.current?.triggerBurst()
    startDwellTimer(s)
  },
  onNavigatingAway: (s, listenSeconds) => {
    recordSignal(s, skipSignalType(listenSeconds))
    cancelDwellTimer()
  },
})
```

---

## Internal mechanisms

### Volume-before-play (fixes bug 4)

Every new player — preloaded or freshly created — gets `setVolume(0)` before `play()` is called. `fadeIn(1000)` then ramps up after the old player's `fadeOut(400)` begins. No window where both players are at full volume.

### Load timeout — 4s (fixes gap 5)

Every `player.play()` call races a 4s rejection:

```ts
const rejectAfter = (ms: number) =>
  new Promise<never>((_, reject) => setTimeout(() => reject(new Error('timeout')), ms))

await Promise.race([player.play(url, false), rejectAfter(4_000)])
```

If the timeout wins, the player is stopped and the attempt is abandoned.

### Retry loop for goNext (fixes bugs 1 and 2)

Up to 3 different stations are attempted. Attempt 0 consumes the preloaded player if ready; attempts 1–2 pick fresh stations.

```
for attempt in 0..2:
  s = stationToLoad ?? (attempt=0 ? consume() : null) ?? pickNext(...)
  player = preloaded?.player ?? new AudioPlayer()
  player.setVolume(0)
  try:
    await Promise.race([player.play(s.url), rejectAfter(4s)])
    // success: crossfade and return
  catch:
    player.stop()
// all failed: keep current station, no feedback
```

### Retry loop for goPrevious (fixes bug 2)

The same station is retried up to 2 times (the user explicitly requested it). If both fail, the current station keeps playing silently.

### transitioningRef guard (fixes bug 2 root cause)

`transitioningRef` is set at the top of the shared advance logic and released in `finally`. Because it is private to the hook, all entry points (`goNext`, `goPrevious`, `loadInitial`, shelf play) share the same guard — no caller can bypass it.

### Stall watchdog — 5s (fixes gap 6)

After a station starts playing, the hook attaches listeners to the current player:

- `waiting` or `stalled` fires → start 5s timer
- `playing` fires → cancel timer (browser recovered on its own)
- Timer fires → call `goNext()` internally + set `stallMessage`

Listeners are torn down and re-attached on each station change. Stale listeners from previous players are never left active.

### Network recovery (fixes gap 7)

The hook listens to `window` `online` event. When it fires:

- If `phase === 'playing'` and player is paused → `player.resume()` (browser silenced it during outage, stream is still alive)
- If `phase === 'playing'` and player is stopped → reconnect to current station (stream died during outage)
- If player is rebuffering (`waiting` state but not fully stopped) → do nothing; let it recover naturally (same logic as iOS `handleNetworkRecovery`)

### Pointer activity / re-prime (replaces reprimeIfNeeded)

The hook exposes `onPointerActivity(station)`. App.tsx calls it from `onPointerDown` and `onPointerMove` on the gesture overlay, same as the current `reprimeIfNeeded` calls. Internally it checks whether the preload slot is empty or expired and re-primes if needed — identical behaviour to the current `usePreload.reprimeIfNeeded`, just private to the hook.

### Player cleanup on unmount (fixes bug 3)

```ts
useEffect(() => () => { playerRef.current?.stop() }, [])
```

Prevents orphaned audio playing after a remount. Also: a `visibilitychange` listener resumes a paused player when the tab regains focus (handles browser audio suspension on tab switch).

---

## AudioPlayer additions

Two new methods added to `AudioPlayer`. No changes to existing methods.

```ts
onStall(cb: () => void): () => void {
  const handler = () => cb()
  this.audio.addEventListener('waiting', handler)
  this.audio.addEventListener('stalled', handler)
  return () => {
    this.audio.removeEventListener('waiting', handler)
    this.audio.removeEventListener('stalled', handler)
  }
}

onRecover(cb: () => void): () => void {
  const handler = () => cb()
  this.audio.addEventListener('playing', handler)
  return () => this.audio.removeEventListener('playing', handler)
}
```

---

## Stall notification UX

When the watchdog auto-advances, `stallMessage` is set to:

> "This station stopped streaming — switched to the next one"

**Placement:** Center of screen, same position as "connecting…". Same typography (system-ui, 12px, 0.2em letter-spacing, uppercase). Color: `rgba(255, 180, 100, 0.5)` (warm amber — distinct from the neutral white of "connecting…").

**Persistence:** Stays until:
- User taps the message (calls `dismissStallMessage()`)
- User navigates (goNext / goPrevious clears it)

NOT cleared by: save, play/pause, share, settings — user may want to read it while interacting with the new station.

**Implementation:**
```tsx
{phase === 'playing' && stallMessage && (
  <div onClick={dismissStallMessage} style={{ position: 'fixed', inset: 0,
    display: 'flex', alignItems: 'center', justifyContent: 'center',
    pointerEvents: 'auto' }}>
    <span style={{ fontFamily: 'system-ui', fontSize: 12, fontWeight: 300,
      letterSpacing: '0.2em', textTransform: 'uppercase',
      color: 'rgba(255,180,100,0.5)' }}>
      {stallMessage}
    </span>
  </div>
)}
```

---

## Testing strategy

`useAudioLifecycle` is the primary unit under test, using `renderHook` + `vitest` with the same `AudioPlayer` mock pattern used elsewhere.

**Required test cases:**

| Case | What is asserted |
|---|---|
| `loadInitial` retries up to 3 stations | succeeds on attempt 3, phase becomes `playing` |
| `loadInitial` exhausts 3 stations | phase returns to `idle`, error message shown |
| `goNext` tries 3 different stations | success on attempt 2, correct station shown |
| `goNext` exhausts 3 stations | current station unchanged, no feedback |
| `goPrevious` retries same station twice | success on retry 2 |
| `goPrevious` exhausts 2 retries | current station unchanged |
| 4s timeout triggers retry | slow `play()` is abandoned, next attempt starts |
| Stall watchdog fires after 5s | `goNext` called, `stallMessage` set |
| Stall recovers before 5s | no auto-advance, `stallMessage` null |
| `dismissStallMessage` | clears `stallMessage` |
| `goNext` clears `stallMessage` | message gone after navigation |
| `online` event resumes paused player | `player.resume()` called |
| `online` event reconnects stopped player | new `play()` started |
| Player stopped on unmount | `player.stop()` called |
| Concurrent `goNext` calls | second call is no-op while first is in flight |
| `onStall` / `onRecover` on AudioPlayer | listeners attach/detach correctly |

**Unaffected tests:** `useNavigation`, `useSwipeGesture`, `useSavedStations`, `useRecommendation`, existing `AudioPlayer` tests (gains 2 new method tests).

`usePreload` tests migrate into `useAudioLifecycle` tests since preload is now an internal.
