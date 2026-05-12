# Audio Fades Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace hard audio cuts with a 1s fade-in on station start and a 400ms crossfade on navigation (old fades out while new fades in).

**Architecture:** `fadeIn`, `fadeOut`, and `cancelFade` are added to `AudioPlayer` using `HTMLAudioElement.volume` animated by a `requestAnimationFrame` loop. The loop records start time from the first rAF timestamp (not `performance.now()`), making it fully testable with a manual rAF mock. `App.tsx` calls these methods at the two points where audio switches — initial load and navigation.

**Tech Stack:** TypeScript, `HTMLAudioElement.volume`, `requestAnimationFrame`, Vitest

---

## File Map

| File | Action | What changes |
|---|---|---|
| `src/audioPlayer.ts` | Modify | Add `fadeRafId` field, private `cancelFade()`, public `fadeIn()`, public `fadeOut()`, update `stop()` |
| `src/audioPlayer.test.ts` | Modify | Add rAF mock stubs + `volume` to Audio mock; add fade tests |
| `src/App.tsx` | Modify | Initial branch: add `fadeIn(1000)`; nav branch: capture old player, call `fadeOut(400)`, remove explicit `stop()`, add `fadeIn(1000)` on new player |

---

## Task 1: `fadeIn`, `cancelFade`, and rAF mock infrastructure

**Files:**
- Modify: `src/audioPlayer.ts`
- Modify: `src/audioPlayer.test.ts`

- [ ] **Step 1: Add rAF mock stubs and `volume` to Audio mock in the test file**

At the top of `src/audioPlayer.test.ts`, before the existing `vi.stubGlobal('AudioContext', ...)` block, add:

```typescript
// rAF mock — manually controlled per test
let mockAudioElement: { volume: number; play: ReturnType<typeof vi.fn>; pause: ReturnType<typeof vi.fn>; src: string; crossOrigin: string; paused: boolean; removeAttribute?: (name: string) => void }

const rafQueue: Map<number, FrameRequestCallback> = new Map()
let rafIdCounter = 0

vi.stubGlobal('requestAnimationFrame', vi.fn((cb: FrameRequestCallback) => {
  const id = ++rafIdCounter
  rafQueue.set(id, cb)
  return id
}))

vi.stubGlobal('cancelAnimationFrame', vi.fn((id: number) => {
  rafQueue.delete(id)
}))

function flushRaf(timestamp: number): void {
  const cbs = [...rafQueue.values()]
  rafQueue.clear()
  cbs.forEach(cb => cb(timestamp))
}
```

Replace the existing `vi.stubGlobal('Audio', ...)` block with one that stores a reference to the mock element and exposes `volume`:

```typescript
vi.stubGlobal('Audio', vi.fn(() => {
  mockAudioElement = {
    play: mockPlay.mockImplementation(() => {
      isPausedValue = false
      return Promise.resolve(undefined)
    }),
    pause: mockPause.mockImplementation(() => {
      isPausedValue = true
    }),
    src: '',
    crossOrigin: '',
    volume: 1,
    get paused() {
      return isPausedValue
    },
  }
  return mockAudioElement
}))
```

In the existing `beforeEach`, add these two resets alongside the existing `vi.clearAllMocks()`:

```typescript
rafQueue.clear()
rafIdCounter = 0
```

- [ ] **Step 2: Add failing tests for `fadeIn`**

Append to `src/audioPlayer.test.ts`, after the existing `describe('pause / resume / isPaused', ...)` block:

```typescript
describe('fadeIn', () => {
  it('sets volume to 0 immediately on call', () => {
    const player = new AudioPlayer()
    player.fadeIn(1000)
    expect(mockAudioElement.volume).toBe(0)
  })

  it('ramps volume to 0.5 at the halfway timestamp', () => {
    const player = new AudioPlayer()
    player.fadeIn(1000)
    flushRaf(0)     // first tick — records startTime = 0, volume = 0
    flushRaf(500)   // halfway — volume = 0.5
    expect(mockAudioElement.volume).toBeCloseTo(0.5)
  })

  it('ramps volume to 1 at the full-duration timestamp', () => {
    const player = new AudioPlayer()
    player.fadeIn(1000)
    flushRaf(0)
    flushRaf(1000)
    expect(mockAudioElement.volume).toBe(1)
  })

  it('cancels a running fadeIn when fadeIn is called again', () => {
    const player = new AudioPlayer()
    player.fadeIn(1000)
    flushRaf(0)    // first fade starts, startTime = 0
    player.fadeIn(1000)   // second call — cancels first, restarts at 0
    flushRaf(500)  // this should advance the SECOND fade, not continue the first
    expect(mockAudioElement.volume).toBeCloseTo(0)  // second fade at t=500 from its own start
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
npm run test:run -- src/audioPlayer.test.ts
```

Expected: FAIL — `player.fadeIn is not a function`

- [ ] **Step 4: Implement `fadeIn` and private `cancelFade` in `src/audioPlayer.ts`**

Add a private field after the existing private fields:

```typescript
private fadeRafId: number | null = null
```

Add the two methods after `isSilent()`:

```typescript
private cancelFade(): void {
  if (this.fadeRafId !== null) {
    cancelAnimationFrame(this.fadeRafId)
    this.fadeRafId = null
  }
}

fadeIn(durationMs: number): void {
  this.cancelFade()
  this.audio.volume = 0
  let startTime: number | null = null
  const tick = (now: number) => {
    if (startTime === null) startTime = now
    const t = Math.min((now - startTime) / durationMs, 1)
    this.audio.volume = t
    if (t < 1) {
      this.fadeRafId = requestAnimationFrame(tick)
    } else {
      this.fadeRafId = null
    }
  }
  this.fadeRafId = requestAnimationFrame(tick)
}
```

- [ ] **Step 5: Run tests to verify fadeIn tests pass**

```bash
npm run test:run -- src/audioPlayer.test.ts
```

Expected: all existing tests pass + all 4 new `fadeIn` tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/audioPlayer.ts src/audioPlayer.test.ts
git commit -m "feat: add AudioPlayer.fadeIn with rAF volume ramp"
```

---

## Task 2: `fadeOut` and update `stop()`

**Files:**
- Modify: `src/audioPlayer.ts`
- Modify: `src/audioPlayer.test.ts`

- [ ] **Step 1: Add failing tests for `fadeOut` and updated `stop()`**

Append to `src/audioPlayer.test.ts`, after the `describe('fadeIn', ...)` block:

```typescript
describe('fadeOut', () => {
  it('ramps volume from 1 to 0.5 at the halfway timestamp', async () => {
    const player = new AudioPlayer()
    mockAudioElement.volume = 1
    const promise = player.fadeOut(400)
    flushRaf(0)    // startTime = 0, startVolume = 1, volume stays 1
    flushRaf(200)  // halfway — volume = 0.5
    expect(mockAudioElement.volume).toBeCloseTo(0.5)  // check BEFORE final flush
    flushRaf(400)  // done — stop() called internally (resets volume to 1)
    await promise
  })

  it('calls stop() after the full duration', async () => {
    const player = new AudioPlayer()
    const promise = player.fadeOut(400)
    flushRaf(0)
    flushRaf(400)
    await promise
    // stop() was called: audio is paused and src cleared
    expect(mockPause).toHaveBeenCalled()
    expect(mockAudioElement.src).toBe('')
  })

  it('cancels a running fadeIn when fadeOut starts', async () => {
    const player = new AudioPlayer()
    player.fadeIn(1000)
    flushRaf(0)                          // fadeIn first tick — startTime = 0
    const promise = player.fadeOut(400)  // cancels the fadeIn, starts fade-out
    flushRaf(0)
    flushRaf(400)
    await promise
    // stop() was called after fade completed
    expect(mockPause).toHaveBeenCalled()
  })
})

describe('stop() with fades', () => {
  it('cancels a running fadeIn and resets volume to 1', () => {
    const player = new AudioPlayer()
    player.fadeIn(1000)
    flushRaf(0)   // first tick — volume still 0, next rAF queued
    player.stop()
    expect(mockAudioElement.volume).toBe(1)
    // no further rAF callbacks should run — queue is cleared
    const volumeBeforeFlush = mockAudioElement.volume
    flushRaf(500)
    expect(mockAudioElement.volume).toBe(volumeBeforeFlush)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/audioPlayer.test.ts
```

Expected: FAIL — `player.fadeOut is not a function`

- [ ] **Step 3: Implement `fadeOut` and update `stop()` in `src/audioPlayer.ts`**

Add `fadeOut` after `fadeIn`:

```typescript
fadeOut(durationMs: number): Promise<void> {
  this.cancelFade()
  const startVolume = this.audio.volume
  let startTime: number | null = null
  return new Promise(resolve => {
    const tick = (now: number) => {
      if (startTime === null) startTime = now
      const t = Math.min((now - startTime) / durationMs, 1)
      this.audio.volume = startVolume * (1 - t)
      if (t < 1) {
        this.fadeRafId = requestAnimationFrame(tick)
      } else {
        this.fadeRafId = null
        this.stop()
        resolve()
      }
    }
    this.fadeRafId = requestAnimationFrame(tick)
  })
}
```

Replace the existing `stop()` method:

```typescript
stop(): void {
  this.cancelFade()
  this.audio.volume = 1
  this.audio.pause()
  this.audio.src = ''
}
```

- [ ] **Step 4: Run the full test suite**

```bash
npm run test:run
```

Expected: all tests pass (96 existing + new fade tests).

- [ ] **Step 5: Commit**

```bash
git add src/audioPlayer.ts src/audioPlayer.test.ts
git commit -m "feat: add AudioPlayer.fadeOut and update stop() to cancel in-progress fades"
```

---

## Task 3: Wire fades into `App.tsx`

**Files:**
- Modify: `src/App.tsx`

No new unit tests — `loadAndPlay` is async/side-effect heavy and not unit-tested. Verified manually via dev server.

- [ ] **Step 1: Add `fadeIn(1000)` in the initial load branch**

In `src/App.tsx`, inside the `isInitial` branch of `loadAndPlay`, replace:

```typescript
        await player.play(s.url, false)
        vizRef.current?.loadStation({ ...s, corsFriendly: false }, player)
```

With:

```typescript
        await player.play(s.url, false)
        player.fadeIn(1000)
        vizRef.current?.loadStation({ ...s, corsFriendly: false }, player)
```

- [ ] **Step 2: Wire crossfade in the navigation branch**

In `src/App.tsx`, inside the navigation `try` block (the block after `transitioningRef.current = true`), replace the entire inner block:

```typescript
      if (!catalogRef.current) catalogRef.current = await loadCatalog()
      const s = stationToLoad ?? pickNext(catalogRef.current, savedStationsRef.current, stationRef.current)
      const player = new AudioPlayer()
      await player.play(s.url, false)   // new stream ready before we switch
      playerRef.current?.stop()
      playerRef.current = player
      vizRef.current?.loadStation({ ...s, corsFriendly: false }, player)
      vizRef.current?.triggerBurst()
      playStartRef.current = Date.now()
      startDwellTimer(s)
      setStation(s)
      return true
```

With:

```typescript
      if (!catalogRef.current) catalogRef.current = await loadCatalog()
      const s = stationToLoad ?? pickNext(catalogRef.current, savedStationsRef.current, stationRef.current)
      const oldPlayer = playerRef.current
      oldPlayer?.fadeOut(400)           // start fade-out immediately; auto-stops after 400ms
      const player = new AudioPlayer()
      await player.play(s.url, false)   // new stream ready before we switch
      playerRef.current = player
      vizRef.current?.loadStation({ ...s, corsFriendly: false }, player)
      vizRef.current?.triggerBurst()
      player.fadeIn(1000)
      playStartRef.current = Date.now()
      startDwellTimer(s)
      setStation(s)
      return true
```

- [ ] **Step 3: Run the full test suite**

```bash
npm run test:run
```

Expected: all tests pass.

- [ ] **Step 4: Verify manually in the dev server**

```bash
npm run dev
```

Open the app. Tap to begin — audio should fade in over ~1s (starts silent, reaches full volume). Navigate with arrow keys or swipe — old stream should fade out over ~0.4s while new stream fades in. Open DevTools → Audio tab to confirm no hard cuts in the waveform.

- [ ] **Step 5: Commit**

```bash
git add src/App.tsx
git commit -m "feat: wire audio crossfade into App — fadeIn on load, fadeOut on nav"
```
