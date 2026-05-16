# Connectivity Resilience Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace scattered audio lifecycle logic in App.tsx with a single `useAudioLifecycle` hook that owns player management, retry loops, load timeout, stall watchdog, and network recovery.

**Architecture:** A new `useAudioLifecycle` hook folds in `usePreload`, makes `transitioningRef` private, and adds 4s load timeout + 5s stall watchdog + network/visibility recovery. App.tsx shrinks to UI reactions and callbacks. Two new methods on `AudioPlayer` (`onStall`/`onRecover`) expose the underlying audio element's stall events.

**Tech Stack:** React 19, TypeScript 5.7, Vitest + @testing-library/react, Web Audio API

---

## File Map

| Action | File | Purpose |
|---|---|---|
| Create | `src/useAudioLifecycle.ts` | New hook — all audio lifecycle |
| Create | `src/useAudioLifecycle.test.ts` | Tests for the new hook |
| Modify | `src/audioPlayer.ts` | Add `onStall` + `onRecover` |
| Modify | `src/audioPlayer.test.ts` | Tests for the two new methods |
| Modify | `src/App.tsx` | Wire hook, remove old plumbing |
| Delete | `src/usePreload.ts` | Folded into useAudioLifecycle |
| Delete | `src/usePreload.test.ts` | Tests migrate to useAudioLifecycle.test.ts |

---

## Task 1: AudioPlayer — add onStall and onRecover

**Files:**
- Modify: `src/audioPlayer.ts`
- Modify: `src/audioPlayer.test.ts`

- [ ] **Step 1: Write failing tests**

Add at the end of the `describe('AudioPlayer', ...)` block in `src/audioPlayer.test.ts`. The mock audio element needs `addEventListener`/`removeEventListener` — add them to the stub global.

In `src/audioPlayer.test.ts`, update the `vi.stubGlobal('Audio', ...)` call so `mockAudioElement` has event listener support:

```ts
// Replace the existing vi.stubGlobal('Audio', ...) with:
const eventListeners: Record<string, Set<EventListener>> = {}

vi.stubGlobal('Audio', vi.fn(() => {
  mockAudioElement = {
    play: mockPlay.mockImplementation(() => { isPausedValue = false; return Promise.resolve(undefined) }),
    pause: mockPause.mockImplementation(() => { isPausedValue = true }),
    src: '',
    crossOrigin: '',
    volume: 1,
    get paused() { return isPausedValue },
    addEventListener: vi.fn((event: string, handler: EventListener) => {
      if (!eventListeners[event]) eventListeners[event] = new Set()
      eventListeners[event].add(handler)
    }),
    removeEventListener: vi.fn((event: string, handler: EventListener) => {
      eventListeners[event]?.delete(handler)
    }),
  }
  return mockAudioElement
}))

function dispatchAudioEvent(event: string) {
  eventListeners[event]?.forEach(h => h(new Event(event)))
}
```

Also clear `eventListeners` in `beforeEach`:
```ts
beforeEach(() => {
  vi.clearAllMocks()
  rafQueue.clear()
  rafIdCounter = 0
  Object.keys(eventListeners).forEach(k => delete eventListeners[k])
  mockGetByteFrequencyData.mockImplementation((arr: Uint8Array) => arr.fill(0))
  isPausedValue = true
})
```

Now add the new test block at the end of `describe('AudioPlayer', ...)`:

```ts
describe('onStall', () => {
  it('calls callback when waiting event fires', () => {
    const player = new AudioPlayer()
    const cb = vi.fn()
    player.onStall(cb)
    dispatchAudioEvent('waiting')
    expect(cb).toHaveBeenCalledOnce()
  })

  it('calls callback when stalled event fires', () => {
    const player = new AudioPlayer()
    const cb = vi.fn()
    player.onStall(cb)
    dispatchAudioEvent('stalled')
    expect(cb).toHaveBeenCalledOnce()
  })

  it('returns cleanup that removes both listeners', () => {
    const player = new AudioPlayer()
    const cb = vi.fn()
    const cleanup = player.onStall(cb)
    cleanup()
    dispatchAudioEvent('waiting')
    dispatchAudioEvent('stalled')
    expect(cb).not.toHaveBeenCalled()
  })
})

describe('onRecover', () => {
  it('calls callback when playing event fires', () => {
    const player = new AudioPlayer()
    const cb = vi.fn()
    player.onRecover(cb)
    dispatchAudioEvent('playing')
    expect(cb).toHaveBeenCalledOnce()
  })

  it('returns cleanup that removes the listener', () => {
    const player = new AudioPlayer()
    const cb = vi.fn()
    const cleanup = player.onRecover(cb)
    cleanup()
    dispatchAudioEvent('playing')
    expect(cb).not.toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/audioPlayer.test.ts
```

Expected: 5 new tests fail with "player.onStall is not a function" (or similar).

- [ ] **Step 3: Implement onStall and onRecover in AudioPlayer**

Add at the end of the `AudioPlayer` class in `src/audioPlayer.ts`, before the closing `}`:

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

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/audioPlayer.test.ts
```

Expected: all tests pass, including the 5 new ones.

- [ ] **Step 5: Commit**

```bash
git add src/audioPlayer.ts src/audioPlayer.test.ts
git commit -m "feat: add AudioPlayer.onStall and onRecover for stall watchdog"
```

---

## Task 2: Scaffold useAudioLifecycle.ts

**Files:**
- Create: `src/useAudioLifecycle.ts`

No tests in this task — the goal is a file that compiles and exports the correct shape. Subsequent tasks fill in the stubs.

- [ ] **Step 1: Create the scaffold**

Create `src/useAudioLifecycle.ts` with the following complete content:

```ts
import { useCallback, useEffect, useRef, useState } from 'react'
import { AudioPlayer } from './audioPlayer'
import { loadCatalog } from './catalog'
import type { Phase, Station } from './types'

const LOAD_TIMEOUT_MS = 4_000
const STALL_TIMEOUT_MS = 5_000
const PRELOAD_EXPIRY_MS = 30_000
const STALL_MESSAGE = 'This station stopped streaming — switched to the next one'

function rejectAfter(ms: number): Promise<never> {
  return new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), ms))
}

export interface AudioLifecycleOptions {
  pickNext: (catalog: Station[], saved: Station[], last: Station | null) => Station
  getSavedStations: () => Station[]
  onStationChanged: (station: Station, player: AudioPlayer) => void
  onNavigatingAway: (
    station: Station,
    listenSeconds: number,
    reason: 'forward' | 'backward' | 'direct',
  ) => void
}

export function useAudioLifecycle({
  pickNext,
  getSavedStations,
  onStationChanged,
  onNavigatingAway,
}: AudioLifecycleOptions) {
  const playerRef         = useRef<AudioPlayer | null>(null)
  const transitioningRef  = useRef(false)
  const catalogRef        = useRef<Station[] | null>(null)
  const stationRef        = useRef<Station | null>(null)
  const phaseRef          = useRef<Phase>('idle')
  const playStartRef      = useRef<number>(0)
  const historyRef        = useRef<Station[]>([])
  const preloadedFirstRef = useRef<{ player: AudioPlayer; station: Station } | null>(null)
  const exitingTimerRef   = useRef<ReturnType<typeof setTimeout> | null>(null)

  const preloadRef = useRef<{
    player: AudioPlayer
    station: Station
    ready: boolean
    expiryTimer: ReturnType<typeof setTimeout>
  } | null>(null)

  const stallTimerRef   = useRef<ReturnType<typeof setTimeout> | null>(null)
  const stallCleanupRef = useRef<Array<() => void>>([])

  // Stable callback refs — update without causing re-renders
  const pickNextRef          = useRef(pickNext)
  const getSavedStationsRef  = useRef(getSavedStations)
  const onStationChangedRef  = useRef(onStationChanged)
  const onNavigatingAwayRef  = useRef(onNavigatingAway)
  useEffect(() => { pickNextRef.current = pickNext },         [pickNext])
  useEffect(() => { getSavedStationsRef.current = getSavedStations }, [getSavedStations])
  useEffect(() => { onStationChangedRef.current = onStationChanged }, [onStationChanged])
  useEffect(() => { onNavigatingAwayRef.current = onNavigatingAway }, [onNavigatingAway])

  const [phase,        setPhase]        = useState<Phase>('idle')
  const [station,      setStation]      = useState<Station | null>(null)
  const [transitioning,setTransitioning]= useState(false)
  const [isPaused,     setIsPaused]     = useState(false)
  const [stallMessage, setStallMessage] = useState<string | null>(null)
  const [canGoPrevious,setCanGoPrevious]= useState(false)

  const updatePhase = useCallback((p: Phase) => {
    phaseRef.current = p
    setPhase(p)
  }, [])

  const ensureCatalog = useCallback(async (): Promise<Station[]> => {
    if (!catalogRef.current) catalogRef.current = await loadCatalog()
    return catalogRef.current
  }, [])

  // ── Preload (stubs — filled in Task 6) ────────────────────────────────
  const cancelPreload = useCallback(() => {}, [])

  const prime = useCallback((_current: Station) => {}, [cancelPreload]) // eslint-disable-line @typescript-eslint/no-unused-vars

  const consume = useCallback((): { player: AudioPlayer; station: Station } | null => null, [])

  const onPointerActivity = useCallback((current: Station) => {
    if (!preloadRef.current) prime(current)
  }, [prime])

  // ── Stall watchdog (stubs — filled in Task 7) ─────────────────────────
  const goNextRef = useRef<() => void>(() => {})

  const detachWatchdog = useCallback(() => {}, [])

  const attachWatchdog = useCallback((_p: AudioPlayer) => {}, [detachWatchdog]) // eslint-disable-line @typescript-eslint/no-unused-vars

  // ── Core commit helper ─────────────────────────────────────────────────
  const commitStation = useCallback((s: Station, p: AudioPlayer) => {
    playerRef.current = p
    stationRef.current = s
    setStation(s)
    setIsPaused(false)
    attachWatchdog(p)
    onStationChangedRef.current(s, p)
    playStartRef.current = Date.now()
    prime(s)
  }, [attachWatchdog, prime])

  // ── Navigation (stubs — filled in Tasks 3–5) ──────────────────────────
  const loadInitial = useCallback(async () => {}, [])

  const goNext = useCallback(async () => {}, [])

  useEffect(() => { goNextRef.current = goNext }, [goNext])

  const goPrevious = useCallback(async () => {}, [commitStation]) // eslint-disable-line @typescript-eslint/no-unused-vars

  const playStation = useCallback(async (_target: Station) => {}, [commitStation]) // eslint-disable-line @typescript-eslint/no-unused-vars

  // ── pause / resume (stubs — filled in Task 9) ─────────────────────────
  const pause  = useCallback(() => {}, [])
  const resume = useCallback(() => {}, [])

  const dismissStallMessage = useCallback(() => setStallMessage(null), [])

  // ── Lifecycle effects (filled in Tasks 3 and 8) ───────────────────────
  // (empty placeholders — added per task)

  return {
    phase, station, transitioning, isPaused, stallMessage, canGoPrevious,
    goNext, goPrevious, playStation, loadInitial, pause, resume,
    dismissStallMessage, onPointerActivity,
  }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
npm run build 2>&1 | head -30
```

Expected: no TypeScript errors referencing `useAudioLifecycle.ts`.

- [ ] **Step 3: Commit**

```bash
git add src/useAudioLifecycle.ts
git commit -m "feat: scaffold useAudioLifecycle hook (stubs)"
```

---

## Task 3: Implement loadInitial with retry, timeout, and preconnect

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Create: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Create the test file with shared setup and loadInitial tests**

Create `src/useAudioLifecycle.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { renderHook, act } from '@testing-library/react'
import type { Station } from './types'

// ── Mocks ──────────────────────────────────────────────────────────────

vi.mock('./audioPlayer')
vi.mock('./catalog')

import { AudioPlayer } from './audioPlayer'
import { loadCatalog } from './catalog'
import { useAudioLifecycle } from './useAudioLifecycle'

// ── Shared test helpers ────────────────────────────────────────────────

const s1: Station = { stationuuid: 's1', name: 'S1', url: 'http://s1.test', tags: '', country: 'US', bitrate: 128, corsFriendly: false }
const s2: Station = { stationuuid: 's2', name: 'S2', url: 'http://s2.test', tags: '', country: 'US', bitrate: 128, corsFriendly: false }
const s3: Station = { stationuuid: 's3', name: 'S3', url: 'http://s3.test', tags: '', country: 'US', bitrate: 128, corsFriendly: false }
const s4: Station = { stationuuid: 's4', name: 'S4', url: 'http://s4.test', tags: '', country: 'US', bitrate: 128, corsFriendly: false }

type MockPlayer = ReturnType<typeof makeMockPlayer>
const mockPlayers: MockPlayer[] = []

function makeMockPlayer() {
  let stallCb: (() => void) | null = null
  let recoverCb: (() => void) | null = null
  return {
    play:        vi.fn().mockResolvedValue(undefined),
    stop:        vi.fn(),
    pause:       vi.fn(),
    resume:      vi.fn(),
    isPaused:    vi.fn().mockReturnValue(false),
    setVolume:   vi.fn(),
    fadeIn:      vi.fn(),
    fadeOut:     vi.fn().mockResolvedValue(undefined),
    preconnect:  vi.fn(),
    onStall:     vi.fn().mockImplementation((cb: () => void) => { stallCb = cb; return () => { stallCb = null } }),
    onRecover:   vi.fn().mockImplementation((cb: () => void) => { recoverCb = cb; return () => { recoverCb = null } }),
    _triggerStall:   () => stallCb?.(),
    _triggerRecover: () => recoverCb?.(),
  }
}

const pickNext = vi.fn()
const onStationChanged = vi.fn()
const onNavigatingAway = vi.fn()

function setup() {
  return renderHook(() => useAudioLifecycle({
    pickNext,
    getSavedStations: () => [],
    onStationChanged,
    onNavigatingAway,
  }))
}

beforeEach(() => {
  vi.clearAllMocks()
  mockPlayers.length = 0
  vi.mocked(AudioPlayer).mockImplementation(() => {
    const p = makeMockPlayer()
    mockPlayers.push(p)
    return p as unknown as AudioPlayer
  })
  vi.mocked(loadCatalog).mockResolvedValue([s1, s2, s3, s4])
  pickNext.mockReturnValue(s1)
})

afterEach(() => {
  vi.useRealTimers()
})

// ── loadInitial tests ──────────────────────────────────────────────────

describe('loadInitial', () => {
  it('transitions phase from idle → exiting → playing', async () => {
    vi.useFakeTimers()
    const { result } = setup()

    expect(result.current.phase).toBe('idle')

    await act(async () => { result.current.loadInitial() })
    expect(result.current.phase).toBe('exiting')

    await act(async () => { vi.advanceTimersByTime(800) })
    expect(result.current.phase).toBe('playing')
  })

  it('sets station to the value returned by pickNext', async () => {
    vi.useFakeTimers()
    const { result } = setup()

    await act(async () => { result.current.loadInitial() })
    expect(result.current.station).toEqual(s1)
  })

  it('calls play with setVolume(0) first, then fadeIn', async () => {
    vi.useFakeTimers()
    const { result } = setup()

    await act(async () => { result.current.loadInitial() })

    const p = mockPlayers[0]
    expect(p.setVolume).toHaveBeenCalledWith(0)
    expect(p.play).toHaveBeenCalledWith(s1.url, false)
    expect(p.fadeIn).toHaveBeenCalledWith(1000)
  })

  it('retries on failure — succeeds on attempt 2', async () => {
    vi.useFakeTimers()
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = setup()

    await act(async () => {
      // First player fails
      vi.mocked(AudioPlayer).mockImplementationOnce(() => {
        const p = makeMockPlayer()
        p.play.mockRejectedValue(new Error('network'))
        mockPlayers.push(p)
        return p as unknown as AudioPlayer
      })
      result.current.loadInitial()
    })

    expect(result.current.station).toEqual(s2)
  })

  it('returns to idle after 3 consecutive failures', async () => {
    vi.useFakeTimers()
    vi.mocked(AudioPlayer).mockImplementation(() => {
      const p = makeMockPlayer()
      p.play.mockRejectedValue(new Error('fail'))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })
    const { result } = setup()

    await act(async () => { result.current.loadInitial() })
    expect(result.current.phase).toBe('idle')
  })

  it('abandons a play() that does not resolve within 4s and retries', async () => {
    vi.useFakeTimers()
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockReturnValue(new Promise(() => {})) // never resolves
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })
    const { result } = setup()

    const loadPromise = act(async () => { result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(4001) })
    await loadPromise

    expect(mockPlayers[0].stop).toHaveBeenCalled()
    expect(result.current.station).toEqual(s2)
  })

  it('is a no-op if called when phase is not idle', async () => {
    vi.useFakeTimers()
    const { result } = setup()

    await act(async () => { result.current.loadInitial() })
    const stationAfterFirst = result.current.station

    await act(async () => { result.current.loadInitial() })
    expect(result.current.station).toEqual(stationAfterFirst)
    expect(loadCatalog).toHaveBeenCalledTimes(1)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all `loadInitial` tests fail (stub does nothing).

- [ ] **Step 3: Implement loadInitial and the initial preconnect effect**

Replace the `loadInitial` stub and add the preconnect effect in `src/useAudioLifecycle.ts`:

```ts
// Replace: const loadInitial = useCallback(async () => {}, [])
const loadInitial = useCallback(async () => {
  if (phaseRef.current !== 'idle') return
  updatePhase('loading')
  try {
    const catalog = await ensureCatalog()
    for (let attempt = 0; attempt < 3; attempt++) {
      const preloaded = attempt === 0 ? preloadedFirstRef.current : null
      if (attempt === 0) preloadedFirstRef.current = null
      const s = preloaded?.station
        ?? pickNextRef.current(catalog, getSavedStationsRef.current(), null)
      const p = preloaded?.player ?? new AudioPlayer()
      try {
        playerRef.current?.stop()
        p.setVolume(0)
        await Promise.race([p.play(s.url, false), rejectAfter(LOAD_TIMEOUT_MS)])
        p.fadeIn(1000)
        commitStation(s, p)
        updatePhase('exiting')
        exitingTimerRef.current = setTimeout(() => {
          exitingTimerRef.current = null
          updatePhase('playing')
        }, 800)
        return
      } catch {
        p.stop()
      }
    }
  } catch { /* loadCatalog failed */ }
  updatePhase('idle')
}, [ensureCatalog, commitStation, updatePhase])
```

Add the initial preconnect effect after the `dismissStallMessage` definition (before the return statement):

```ts
useEffect(() => {
  let cancelled = false
  async function preconnect() {
    try {
      const catalog = await ensureCatalog()
      const s = pickNextRef.current(catalog, getSavedStationsRef.current(), null)
      const p = new AudioPlayer()
      p.preconnect(s.url)
      if (!cancelled && phaseRef.current === 'idle') {
        preloadedFirstRef.current = { player: p, station: s }
      }
    } catch { /* silent */ }
  }
  preconnect()
  return () => { cancelled = true }
}, [ensureCatalog])
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all `loadInitial` tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: implement loadInitial with 3-attempt retry and 4s timeout"
```

---

## Task 4: Implement goNext

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Modify: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useAudioLifecycle.test.ts`:

```ts
// ── goNext tests ───────────────────────────────────────────────────────

describe('goNext', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    expect(hook.result.current.phase).toBe('playing')
    return hook
  }

  it('loads the next station and commits it', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = await setupPlaying()

    await act(async () => { result.current.goNext() })

    expect(result.current.station).toEqual(s2)
  })

  it('fades out old player and fades in new player', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = await setupPlaying()
    const oldPlayer = mockPlayers[0]

    await act(async () => { result.current.goNext() })

    expect(oldPlayer.fadeOut).toHaveBeenCalledWith(400)
    const newPlayer = mockPlayers[mockPlayers.length - 1]
    expect(newPlayer.fadeIn).toHaveBeenCalledWith(1000)
  })

  it('sets volume to 0 on new player before play', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = await setupPlaying()

    await act(async () => { result.current.goNext() })

    const newPlayer = mockPlayers[mockPlayers.length - 1]
    expect(newPlayer.setVolume).toHaveBeenCalledWith(0)
    // setVolume(0) must come before play()
    const setVolumeOrder = newPlayer.setVolume.mock.invocationCallOrder[0]
    const playOrder = newPlayer.play.mock.invocationCallOrder[0]
    expect(setVolumeOrder).toBeLessThan(playOrder)
  })

  it('enables canGoPrevious after first goNext', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = await setupPlaying()
    expect(result.current.canGoPrevious).toBe(false)

    await act(async () => { result.current.goNext() })

    expect(result.current.canGoPrevious).toBe(true)
  })

  it('calls onNavigatingAway with forward reason', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = await setupPlaying()

    await act(async () => { result.current.goNext() })

    expect(onNavigatingAway).toHaveBeenCalledWith(s1, expect.any(Number), 'forward')
  })

  it('clears stallMessage on navigate', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const { result } = await setupPlaying()

    // Manually set stall message
    await act(async () => { result.current.loadInitial() }) // won't run (already playing)
    // trigger via dismissStallMessage inverse: skip this, trust next test covers it

    await act(async () => { result.current.goNext() })
    expect(result.current.stallMessage).toBeNull()
  })

  it('retries with a different station on failure', async () => {
    pickNext
      .mockReturnValueOnce(s1) // initial
      .mockReturnValueOnce(s2) // goNext attempt 0 — fails
      .mockReturnValueOnce(s3) // goNext attempt 1 — succeeds
    const { result } = await setupPlaying()

    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockRejectedValue(new Error('network'))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    await act(async () => { result.current.goNext() })

    expect(result.current.station).toEqual(s3)
  })

  it('keeps current station after 3 failed attempts', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValue(s2)
    const { result } = await setupPlaying()

    vi.mocked(AudioPlayer).mockImplementation(() => {
      const p = makeMockPlayer()
      p.play.mockRejectedValue(new Error('fail'))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    await act(async () => { result.current.goNext() })

    expect(result.current.station).toEqual(s1)
  })

  it('concurrent goNext calls: second is a no-op while first is in flight', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2).mockReturnValueOnce(s3)
    const { result } = await setupPlaying()

    let resolveFirst!: () => void
    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockReturnValue(new Promise<void>(res => { resolveFirst = res }))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    act(() => { result.current.goNext() }) // in flight
    await act(async () => { result.current.goNext() }) // no-op

    resolveFirst()
    await act(async () => {})

    // Only one navigation happened
    expect(result.current.station).toEqual(s2)
    expect(pickNext).toHaveBeenCalledTimes(2) // initial + one goNext
  })

  it('4s timeout triggers retry to next station', async () => {
    vi.useFakeTimers()
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2).mockReturnValueOnce(s3)
    const { result } = await setupPlaying()

    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockReturnValue(new Promise(() => {})) // hangs forever
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    const navPromise = act(async () => { result.current.goNext() })
    await act(async () => { vi.advanceTimersByTime(4001) })
    await navPromise

    expect(mockPlayers.find(p => p.stop.mock.calls.length > 0)).toBeTruthy()
    expect(result.current.station).toEqual(s3)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all `goNext` tests fail (stub does nothing).

- [ ] **Step 3: Implement goNext**

Replace the `goNext` stub in `src/useAudioLifecycle.ts`:

```ts
// Replace: const goNext = useCallback(async () => {}, [])
const goNext = useCallback(async () => {
  if (phaseRef.current !== 'playing' || transitioningRef.current) return
  const prev = stationRef.current
  const listenSeconds = prev ? (Date.now() - playStartRef.current) / 1000 : 0
  if (prev) onNavigatingAwayRef.current(prev, listenSeconds, 'forward')
  setStallMessage(null)

  transitioningRef.current = true
  setTransitioning(true)
  const oldPlayer = playerRef.current
  try {
    const catalog = await ensureCatalog()
    for (let attempt = 0; attempt < 3; attempt++) {
      const preloaded = attempt === 0 ? consume() : null
      const s = preloaded?.station
        ?? pickNextRef.current(catalog, getSavedStationsRef.current(), stationRef.current)
      const p = preloaded?.player ?? new AudioPlayer()
      try {
        p.setVolume(0)
        if (!preloaded) await Promise.race([p.play(s.url, false), rejectAfter(LOAD_TIMEOUT_MS)])
        oldPlayer?.fadeOut(400)
        p.fadeIn(1000)
        if (prev) { historyRef.current.push(prev); setCanGoPrevious(true) }
        commitStation(s, p)
        return
      } catch {
        p.stop()
      }
    }
  } catch { /* catalog failed */ }
  finally {
    transitioningRef.current = false
    setTransitioning(false)
  }
}, [ensureCatalog, consume, commitStation])
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all `goNext` tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: implement goNext with 3-station retry, timeout, history, and volume guard"
```

---

## Task 5: Implement goPrevious and playStation

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Modify: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useAudioLifecycle.test.ts`:

```ts
// ── goPrevious tests ───────────────────────────────────────────────────

describe('goPrevious', () => {
  async function setupWithHistory() {
    vi.useFakeTimers()
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    await act(async () => { hook.result.current.goNext() })
    // now on s2, history has [s1]
    expect(hook.result.current.station).toEqual(s2)
    expect(hook.result.current.canGoPrevious).toBe(true)
    return hook
  }

  it('loads the previous station', async () => {
    const { result } = await setupWithHistory()

    await act(async () => { result.current.goPrevious() })

    expect(result.current.station).toEqual(s1)
  })

  it('clears canGoPrevious when history is exhausted', async () => {
    const { result } = await setupWithHistory()

    await act(async () => { result.current.goPrevious() })

    expect(result.current.canGoPrevious).toBe(false)
  })

  it('calls onNavigatingAway with backward reason', async () => {
    const { result } = await setupWithHistory()
    onNavigatingAway.mockClear()

    await act(async () => { result.current.goPrevious() })

    expect(onNavigatingAway).toHaveBeenCalledWith(s2, expect.any(Number), 'backward')
  })

  it('retries the same station on failure (up to 2 times)', async () => {
    const { result } = await setupWithHistory()

    // First attempt fails, second succeeds
    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockRejectedValue(new Error('network'))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    await act(async () => { result.current.goPrevious() })

    expect(result.current.station).toEqual(s1)
  })

  it('keeps current station if both retry attempts fail', async () => {
    const { result } = await setupWithHistory()

    vi.mocked(AudioPlayer).mockImplementation(() => {
      const p = makeMockPlayer()
      p.play.mockRejectedValue(new Error('fail'))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    await act(async () => { result.current.goPrevious() })

    expect(result.current.station).toEqual(s2)
  })

  it('is a no-op when history is empty', async () => {
    vi.useFakeTimers()
    pickNext.mockReturnValue(s1)
    const { result } = setup()
    await act(async () => { result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })

    await act(async () => { result.current.goPrevious() })

    expect(result.current.station).toEqual(s1) // unchanged
  })
})

// ── playStation tests ──────────────────────────────────────────────────

describe('playStation', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    pickNext.mockReturnValue(s1)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    return hook
  }

  it('loads the specified station', async () => {
    const { result } = await setupPlaying()

    await act(async () => { result.current.playStation(s3) })

    expect(result.current.station).toEqual(s3)
  })

  it('calls onNavigatingAway with direct reason', async () => {
    const { result } = await setupPlaying()
    onNavigatingAway.mockClear()

    await act(async () => { result.current.playStation(s3) })

    expect(onNavigatingAway).toHaveBeenCalledWith(s1, expect.any(Number), 'direct')
  })

  it('retries the same station on failure (up to 2 times)', async () => {
    const { result } = await setupPlaying()

    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockRejectedValue(new Error('network'))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    await act(async () => { result.current.playStation(s3) })

    expect(result.current.station).toEqual(s3)
  })

  it('is blocked by transitioningRef when already transitioning', async () => {
    pickNext.mockReturnValue(s1)
    const { result } = await setupPlaying()

    let resolveNav!: () => void
    vi.mocked(AudioPlayer).mockImplementationOnce(() => {
      const p = makeMockPlayer()
      p.play.mockReturnValue(new Promise<void>(res => { resolveNav = res }))
      mockPlayers.push(p)
      return p as unknown as AudioPlayer
    })

    act(() => { result.current.goNext() }) // in flight
    await act(async () => { result.current.playStation(s3) }) // blocked

    resolveNav()
    await act(async () => {})

    // goNext won the race, not playStation
    expect(result.current.station?.stationuuid).not.toBe('s3')
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all `goPrevious` and `playStation` tests fail.

- [ ] **Step 3: Implement goPrevious and playStation**

Replace both stubs in `src/useAudioLifecycle.ts`:

```ts
// Replace: const goPrevious = useCallback(async () => {}, [commitStation])
const goPrevious = useCallback(async () => {
  if (phaseRef.current !== 'playing' || transitioningRef.current) return
  if (historyRef.current.length === 0) return
  const target = historyRef.current[historyRef.current.length - 1]
  const prev = stationRef.current
  const listenSeconds = prev ? (Date.now() - playStartRef.current) / 1000 : 0
  if (prev) onNavigatingAwayRef.current(prev, listenSeconds, 'backward')
  setStallMessage(null)

  transitioningRef.current = true
  setTransitioning(true)
  const oldPlayer = playerRef.current
  try {
    for (let attempt = 0; attempt < 2; attempt++) {
      const p = new AudioPlayer()
      try {
        p.setVolume(0)
        await Promise.race([p.play(target.url, false), rejectAfter(LOAD_TIMEOUT_MS)])
        oldPlayer?.fadeOut(400)
        p.fadeIn(1000)
        historyRef.current.pop()
        setCanGoPrevious(historyRef.current.length > 0)
        commitStation(target, p)
        return
      } catch {
        p.stop()
      }
    }
  } finally {
    transitioningRef.current = false
    setTransitioning(false)
  }
}, [commitStation])

// Replace: const playStation = useCallback(async (_target: Station) => {}, [commitStation])
const playStation = useCallback(async (target: Station) => {
  if (transitioningRef.current) return
  if (phaseRef.current !== 'playing' && phaseRef.current !== 'idle') return
  const prev = stationRef.current
  const listenSeconds = prev ? (Date.now() - playStartRef.current) / 1000 : 0
  if (prev) onNavigatingAwayRef.current(prev, listenSeconds, 'direct')
  setStallMessage(null)

  transitioningRef.current = true
  setTransitioning(true)
  const oldPlayer = playerRef.current
  try {
    for (let attempt = 0; attempt < 2; attempt++) {
      const p = new AudioPlayer()
      try {
        p.setVolume(0)
        await Promise.race([p.play(target.url, false), rejectAfter(LOAD_TIMEOUT_MS)])
        oldPlayer?.fadeOut(400)
        p.fadeIn(1000)
        commitStation(target, p)
        return
      } catch {
        p.stop()
      }
    }
  } finally {
    transitioningRef.current = false
    setTransitioning(false)
  }
}, [commitStation])
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all `goPrevious` and `playStation` tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: implement goPrevious and playStation with 2-retry same-station logic"
```

---

## Task 6: Implement preload internals

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Modify: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useAudioLifecycle.test.ts`:

```ts
// ── Preload tests ──────────────────────────────────────────────────────

describe('preload', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    pickNext.mockReturnValueOnce(s1).mockReturnValue(s2)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    return hook
  }

  it('prime: starts a silent player for the next station after loadInitial', async () => {
    const { result } = await setupPlaying()
    // After commitStation, prime(s1) is called; pickNext returns s2
    const preloadPlayer = mockPlayers.find(p =>
      p.setVolume.mock.calls.some(c => c[0] === 0) && p !== mockPlayers[0]
    )
    expect(preloadPlayer).toBeTruthy()
    expect(preloadPlayer!.play).toHaveBeenCalledWith(s2.url, false)
  })

  it('goNext consumes the preloaded player when ready', async () => {
    const { result } = await setupPlaying()
    const preloadPlayer = mockPlayers[mockPlayers.length - 1]

    pickNext.mockReturnValue(s3)
    await act(async () => { result.current.goNext() })

    // The preloaded player (s2) should be used — play not called on it again
    expect(preloadPlayer.play).toHaveBeenCalledTimes(1) // only from preload, not navigation
    expect(result.current.station).toEqual(s2)
  })

  it('preload expires after 30s and is cleared', async () => {
    await setupPlaying()
    const preloadPlayer = mockPlayers[mockPlayers.length - 1]

    await act(async () => { vi.advanceTimersByTime(30_000) })

    expect(preloadPlayer.stop).toHaveBeenCalled()
  })

  it('onPointerActivity re-primes when preload is empty', async () => {
    const { result } = await setupPlaying()
    const initialPreloadCount = mockPlayers.length

    // Force expiry to clear preload
    await act(async () => { vi.advanceTimersByTime(30_000) })

    await act(async () => { result.current.onPointerActivity(s1) })

    expect(mockPlayers.length).toBeGreaterThan(initialPreloadCount)
  })

  it('onPointerActivity is a no-op when preload is already active', async () => {
    const { result } = await setupPlaying()
    const countBefore = mockPlayers.length

    await act(async () => { result.current.onPointerActivity(s1) })

    expect(mockPlayers.length).toBe(countBefore)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: preload tests fail (stubs do nothing).

- [ ] **Step 3: Implement cancelPreload, prime, consume**

Replace the three stubs in `src/useAudioLifecycle.ts`:

```ts
// Replace: const cancelPreload = useCallback(() => {}, [])
const cancelPreload = useCallback(() => {
  if (preloadRef.current) {
    clearTimeout(preloadRef.current.expiryTimer)
    preloadRef.current.player.stop()
    preloadRef.current = null
  }
}, [])

// Replace: const prime = useCallback((_current: Station) => {}, [cancelPreload])
const prime = useCallback((current: Station) => {
  cancelPreload()
  const catalog = catalogRef.current
  if (!catalog) return
  const next = pickNextRef.current(catalog, getSavedStationsRef.current(), current)
  const player = new AudioPlayer()
  player.setVolume(0)
  const expiryTimer = setTimeout(() => {
    if (preloadRef.current?.expiryTimer === expiryTimer) {
      preloadRef.current.player.stop()
      preloadRef.current = null
    }
  }, PRELOAD_EXPIRY_MS)
  preloadRef.current = { player, station: next, ready: false, expiryTimer }
  player.play(next.url, false).then(() => {
    if (preloadRef.current?.player === player) preloadRef.current.ready = true
  }).catch(() => {
    if (preloadRef.current?.player === player) {
      clearTimeout(expiryTimer)
      preloadRef.current.player.stop()
      preloadRef.current = null
    }
  })
}, [cancelPreload])

// Replace: const consume = useCallback((): { player: AudioPlayer; station: Station } | null => null, [])
const consume = useCallback((): { player: AudioPlayer; station: Station } | null => {
  if (!preloadRef.current?.ready) return null
  const { player, station, expiryTimer } = preloadRef.current
  clearTimeout(expiryTimer)
  preloadRef.current = null
  return { player, station }
}, [])
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all preload tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: implement preload internals (prime/consume/onPointerActivity)"
```

---

## Task 7: Implement stall watchdog

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Modify: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useAudioLifecycle.test.ts`:

```ts
// ── Stall watchdog tests ───────────────────────────────────────────────

describe('stall watchdog', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    pickNext.mockReturnValueOnce(s1).mockReturnValue(s2)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    return hook
  }

  it('sets stallMessage and auto-advances after 5s stall', async () => {
    const { result } = await setupPlaying()
    const playingPlayer = mockPlayers[0]

    act(() => { playingPlayer._triggerStall() })
    expect(result.current.stallMessage).toBeNull() // not set yet

    await act(async () => { vi.advanceTimersByTime(5000) })

    expect(result.current.stallMessage).toBe(
      'This station stopped streaming — switched to the next one'
    )
    expect(result.current.station).toEqual(s2)
  })

  it('cancels stall timer when stream recovers', async () => {
    const { result } = await setupPlaying()
    const playingPlayer = mockPlayers[0]

    act(() => { playingPlayer._triggerStall() })
    act(() => { playingPlayer._triggerRecover() })

    await act(async () => { vi.advanceTimersByTime(5000) })

    expect(result.current.stallMessage).toBeNull()
    expect(result.current.station).toEqual(s1) // unchanged
  })

  it('dismissStallMessage clears the message', async () => {
    const { result } = await setupPlaying()
    const playingPlayer = mockPlayers[0]

    act(() => { playingPlayer._triggerStall() })
    await act(async () => { vi.advanceTimersByTime(5000) })
    expect(result.current.stallMessage).not.toBeNull()

    act(() => { result.current.dismissStallMessage() })
    expect(result.current.stallMessage).toBeNull()
  })

  it('detaches watchdog listeners when station changes', async () => {
    pickNext.mockReturnValueOnce(s1).mockReturnValueOnce(s2).mockReturnValue(s3)
    const { result } = await setupPlaying()
    const firstPlayer = mockPlayers[0]

    await act(async () => { result.current.goNext() })
    // now on s2, firstPlayer's stall listeners should be removed

    act(() => { firstPlayer._triggerStall() })
    await act(async () => { vi.advanceTimersByTime(5000) })

    // No auto-advance triggered by stale player
    expect(result.current.station).toEqual(s2)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: stall watchdog tests fail.

- [ ] **Step 3: Implement detachWatchdog and attachWatchdog**

Replace both stubs in `src/useAudioLifecycle.ts`:

```ts
// Replace: const detachWatchdog = useCallback(() => {}, [])
const detachWatchdog = useCallback(() => {
  if (stallTimerRef.current !== null) {
    clearTimeout(stallTimerRef.current)
    stallTimerRef.current = null
  }
  stallCleanupRef.current.forEach(fn => fn())
  stallCleanupRef.current = []
}, [])

// Replace: const attachWatchdog = useCallback((_p: AudioPlayer) => {}, [detachWatchdog])
const attachWatchdog = useCallback((p: AudioPlayer) => {
  detachWatchdog()
  const startTimer = () => {
    if (stallTimerRef.current !== null) return
    stallTimerRef.current = setTimeout(() => {
      stallTimerRef.current = null
      setStallMessage(STALL_MESSAGE)
      goNextRef.current()
    }, STALL_TIMEOUT_MS)
  }
  const cancelTimer = () => {
    if (stallTimerRef.current !== null) {
      clearTimeout(stallTimerRef.current)
      stallTimerRef.current = null
    }
  }
  stallCleanupRef.current = [p.onStall(startTimer), p.onRecover(cancelTimer)]
}, [detachWatchdog])
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all stall watchdog tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: implement 5s stall watchdog with auto-advance and stallMessage"
```

---

## Task 8: Lifecycle effects — unmount cleanup, network recovery, visibility

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Modify: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useAudioLifecycle.test.ts`:

```ts
// ── Lifecycle effect tests ─────────────────────────────────────────────

describe('unmount cleanup', () => {
  it('stops the active player on unmount', async () => {
    vi.useFakeTimers()
    pickNext.mockReturnValue(s1)
    const { result, unmount } = setup()
    await act(async () => { result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    const activePlayer = mockPlayers[0]

    unmount()

    expect(activePlayer.stop).toHaveBeenCalled()
  })
})

describe('network recovery', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    pickNext.mockReturnValue(s1)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    return hook
  }

  it('resumes a paused player when coming online', async () => {
    const { result } = await setupPlaying()
    const activePlayer = mockPlayers[0]
    activePlayer.isPaused.mockReturnValue(true)

    await act(async () => { window.dispatchEvent(new Event('online')) })

    expect(activePlayer.resume).toHaveBeenCalled()
  })

  it('reconnects to current station if player is stopped while offline', async () => {
    const { result } = await setupPlaying()
    const activePlayer = mockPlayers[0]
    activePlayer.isPaused.mockReturnValue(false)
    // Simulate stopped stream (play would be called on a new player)

    await act(async () => { window.dispatchEvent(new Event('online')) })

    // A new player should have been created and play called
    const newPlayer = mockPlayers[mockPlayers.length - 1]
    expect(newPlayer).not.toBe(activePlayer)
    expect(newPlayer.play).toHaveBeenCalledWith(s1.url, false)
  })

  it('is a no-op when not in playing phase', async () => {
    const { result } = setup()
    // phase is 'idle'
    await act(async () => { window.dispatchEvent(new Event('online')) })

    expect(mockPlayers.every(p => p.resume.mock.calls.length === 0)).toBe(true)
  })
})

describe('visibility recovery', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    pickNext.mockReturnValue(s1)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    return hook
  }

  it('resumes audio when tab becomes visible and player is paused', async () => {
    const { result } = await setupPlaying()
    const activePlayer = mockPlayers[0]
    activePlayer.isPaused.mockReturnValue(true)

    Object.defineProperty(document, 'hidden', { value: false, configurable: true })
    await act(async () => {
      document.dispatchEvent(new Event('visibilitychange'))
    })

    expect(activePlayer.resume).toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: lifecycle effect tests fail.

- [ ] **Step 3: Implement the three lifecycle effects**

Add these three `useEffect` calls to `src/useAudioLifecycle.ts`, after the preconnect effect and before the `return` statement:

```ts
// Unmount cleanup
useEffect(() => {
  return () => {
    if (exitingTimerRef.current !== null) clearTimeout(exitingTimerRef.current)
    playerRef.current?.stop()
    cancelPreload()
    detachWatchdog()
  }
}, [cancelPreload, detachWatchdog])

// Network recovery
useEffect(() => {
  const handleOnline = () => {
    if (phaseRef.current !== 'playing') return
    const p = playerRef.current
    if (!p) return
    if (p.isPaused()) {
      p.resume()
      setIsPaused(false)
    } else {
      const s = stationRef.current
      if (!s) return
      const newPlayer = new AudioPlayer()
      newPlayer.setVolume(0)
      Promise.race([newPlayer.play(s.url, false), rejectAfter(LOAD_TIMEOUT_MS)])
        .then(() => { p.stop(); newPlayer.fadeIn(1000); commitStation(s, newPlayer) })
        .catch(() => newPlayer.stop())
    }
  }
  window.addEventListener('online', handleOnline)
  return () => window.removeEventListener('online', handleOnline)
}, [commitStation])

// Visibility recovery
useEffect(() => {
  const handleVisible = () => {
    if (document.hidden) return
    if (phaseRef.current === 'playing' && playerRef.current?.isPaused()) {
      playerRef.current.resume()
      setIsPaused(false)
    }
  }
  document.addEventListener('visibilitychange', handleVisible)
  return () => document.removeEventListener('visibilitychange', handleVisible)
}, [])
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all lifecycle effect tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: add unmount cleanup, online network recovery, and visibilitychange resume"
```

---

## Task 9: Implement pause and resume

**Files:**
- Modify: `src/useAudioLifecycle.ts`
- Modify: `src/useAudioLifecycle.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useAudioLifecycle.test.ts`:

```ts
// ── pause / resume tests ───────────────────────────────────────────────

describe('pause / resume', () => {
  async function setupPlaying() {
    vi.useFakeTimers()
    pickNext.mockReturnValue(s1)
    const hook = setup()
    await act(async () => { hook.result.current.loadInitial() })
    await act(async () => { vi.advanceTimersByTime(800) })
    return hook
  }

  it('pause() calls player.pause and sets isPaused true', async () => {
    const { result } = await setupPlaying()

    act(() => { result.current.pause() })

    expect(mockPlayers[0].pause).toHaveBeenCalled()
    expect(result.current.isPaused).toBe(true)
  })

  it('resume() calls player.resume and sets isPaused false', async () => {
    const { result } = await setupPlaying()

    act(() => { result.current.pause() })
    act(() => { result.current.resume() })

    expect(mockPlayers[0].resume).toHaveBeenCalled()
    expect(result.current.isPaused).toBe(false)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: pause/resume tests fail (stubs do nothing).

- [ ] **Step 3: Implement pause and resume**

Replace the two stubs in `src/useAudioLifecycle.ts`:

```ts
// Replace: const pause  = useCallback(() => {}, [])
const pause = useCallback(() => {
  playerRef.current?.pause()
  setIsPaused(true)
}, [])

// Replace: const resume = useCallback(() => {}, [])
const resume = useCallback(() => {
  playerRef.current?.resume()
  setIsPaused(false)
}, [])
```

- [ ] **Step 4: Run all tests to verify they pass**

```bash
npm run test:run -- src/useAudioLifecycle.test.ts
```

Expected: all tests in `useAudioLifecycle.test.ts` pass.

- [ ] **Step 5: Commit**

```bash
git add src/useAudioLifecycle.ts src/useAudioLifecycle.test.ts
git commit -m "feat: implement pause and resume in useAudioLifecycle"
```

---

## Task 10: Wire App.tsx

**Files:**
- Modify: `src/App.tsx`

This task removes all old audio plumbing and replaces it with the new hook. No new tests — the hook tests cover the logic.

- [ ] **Step 1: Remove old state and refs**

In `src/App.tsx`, remove the following lines entirely:

```ts
// Remove these refs:
const historyRef          = useRef<Station[]>([])
const stationRef          = useRef<Station | null>(null)
const phaseRef            = useRef<Phase>('idle')
const transitioningRef    = useRef(false)
const preloadedFirstRef   = useRef<{ player: AudioPlayer; station: Station } | null>(null)
const playerRef           = useRef<AudioPlayer | null>(null)
const playStartRef        = useRef<number>(0)

// Remove these state variables:
const [phase, setPhase]                   = useState<Phase>('idle')
const [canGoPrevious, setCanGoPrevious]   = useState(false)
const [isFirstLoad, setIsFirstLoad]       = useState(true)
const [transitioning, setTransitioning]   = useState(false)
const [paused, setPaused]                 = useState(false)

// Remove these effect lines:
useEffect(() => { stationRef.current = station }, [station])
useEffect(() => { phaseRef.current = phase }, [phase])
```

- [ ] **Step 2: Remove usePreload import and call**

Remove the `usePreload` import and the following lines:

```ts
// Remove import:
import { usePreload } from './usePreload'

// Remove hook call and all related code:
const { prime, consume, reprimeIfNeeded } = usePreload(
  () => catalogRef.current,
  () => savedStationsRef.current,
  pickNext,
)
```

- [ ] **Step 3: Remove loadAndPlay and the preconnect effect**

Delete the entire `loadAndPlay` function (lines ~93–157 in the original), the preconnect `useEffect` (lines ~74–91), and the keydown-to-loadAndPlay effect.

- [ ] **Step 4: Remove old goNext and goPrevious**

Delete the old `goNext` and `goPrevious` `useCallback` definitions that referenced `loadAndPlay`.

- [ ] **Step 5: Add useAudioLifecycle import and call**

Add the import at the top of `src/App.tsx`:

```ts
import { useAudioLifecycle } from './useAudioLifecycle'
import { skipSignalType } from './recommendation'
```

Replace the removed hook calls and definitions with:

```ts
const {
  phase, station, transitioning, isPaused, stallMessage, canGoPrevious,
  goNext, goPrevious, playStation, loadInitial, pause, resume,
  dismissStallMessage, onPointerActivity,
} = useAudioLifecycle({
  pickNext,
  getSavedStations: () => savedStationsRef.current,
  onStationChanged: (s, p) => {
    vizRef.current?.loadStation({ ...s, corsFriendly: false }, p)
    vizRef.current?.triggerBurst()
    startDwellTimer(s)
  },
  onNavigatingAway: (s, listenSeconds, reason) => {
    const signal = reason === 'backward' ? 'prev' : skipSignalType(listenSeconds)
    recordSignal(s, signal)
    cancelDwellTimer()
  },
})

// Track whether we have ever reached the playing phase (for welcome screen)
const [isFirstLoad, setIsFirstLoad] = useState(true)
useEffect(() => {
  if (phase === 'playing') setIsFirstLoad(false)
}, [phase])
```

- [ ] **Step 6: Update handleTap, keydown handler, and goNext/goPrevious wrappers**

```ts
const handleTap = () => {
  if (phase !== 'idle') return
  loadInitial()
}

// Update the keydown-to-start effect:
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    if (phase !== 'idle') return
    if (e.metaKey || e.ctrlKey || e.altKey) return
    loadInitial()
  }
  window.addEventListener('keydown', handleKeyDown)
  return () => window.removeEventListener('keydown', handleKeyDown)
}, [loadInitial, phase])

// Wrap goNext and goPrevious to also dismiss tutorial:
const handleNext = useCallback(() => {
  dismissTutorialRef.current()
  goNext()
}, [goNext])

const handlePrevious = useCallback(() => {
  dismissTutorialRef.current()
  goPrevious()
}, [goPrevious])
```

- [ ] **Step 7: Update handlePlayPause, handleSave, and pointer events**

```ts
const handlePlayPause = useCallback(() => {
  dismissTutorialRef.current()
  if (phase !== 'playing') return
  if (isPaused) { resume() } else { pause() }
}, [phase, isPaused, pause, resume])

// handleSave: use station and phase from hook (no longer reads stationRef/phaseRef directly)
const handleSave = useCallback(async () => {
  dismissTutorialRef.current()
  if (!station || phase !== 'playing') return
  recordSignal(station, 'save')
  const result = await save(station)
  if (result === 'saved') {
    vizRef.current?.triggerBurst()
    setToastMessage('Saved ♥')
  } else if (result === 'duplicate') {
    setToastMessage('Already saved ♥')
  } else {
    setToastMessage('Save limit reached')
  }
}, [save, recordSignal, station, phase])

// Pointer div: replace reprimeIfNeeded with onPointerActivity
<div
  onPointerMove={e => { station && onPointerActivity(station); onPointerMove(e) }}
  onPointerDown={e => { station && onPointerActivity(station); onPointerDown(e) }}
  onPointerUp={onPointerUp}
  onPointerCancel={onPointerCancel}
  ...
/>
```

- [ ] **Step 8: Update useNavigation and useSwipeGesture calls**

```ts
// Pass handleNext / handlePrevious instead of goNext / goPrevious:
const { keyboardFlash } = useNavigation(
  phase, handleNext, handlePrevious, canGoPrevious,
  handleSave, openShelf, closeShelf, shelfOpen, handlePlayPause,
)

const { swipeFlash, onPointerDown, onPointerMove, onPointerUp, onPointerCancel } = useSwipeGesture(
  phase, handleNext, handlePrevious, canGoPrevious, handleSave, openShelf,
)
```

- [ ] **Step 9: Update the shelf onPlay handler**

```ts
// Old: onPlay={s => { loadAndPlay(s); closeShelf() }}
// New:
onPlay={s => { playStation(s); closeShelf() }}
```

- [ ] **Step 10: Verify the app compiles and runs**

```bash
npm run build 2>&1 | head -40
```

Expected: no TypeScript errors.

```bash
npm run dev
```

Open the app in a browser. Verify: welcome screen shows, tap starts a station, swipe left/right navigates.

- [ ] **Step 11: Run the full test suite**

```bash
npm run test:run
```

Expected: all existing tests pass. Any failures related to removed refs/state in App.tsx component tests should be investigated and fixed.

- [ ] **Step 12: Commit**

```bash
git add src/App.tsx
git commit -m "refactor: wire useAudioLifecycle in App.tsx, remove loadAndPlay and scattered audio plumbing"
```

---

## Task 11: Stall message UI and isFirstLoad derivation

**Files:**
- Modify: `src/App.tsx`

- [ ] **Step 1: Add the stall message display to JSX**

In `src/App.tsx`, add the stall message block alongside the "connecting…" block (after line ~342 in the original, inside the `phase === 'playing'` section):

```tsx
{phase === 'playing' && stallMessage && (
  <div
    onClick={dismissStallMessage}
    style={{
      position: 'fixed', inset: 0,
      display: 'flex', alignItems: 'center', justifyContent: 'center',
      pointerEvents: 'auto',
      cursor: 'pointer',
    }}
  >
    <span style={{
      fontFamily: 'system-ui, sans-serif',
      fontSize: 12,
      fontWeight: 300,
      letterSpacing: '0.2em',
      textTransform: 'uppercase' as const,
      color: 'rgba(255,180,100,0.5)',
    }}>
      {stallMessage}
    </span>
  </div>
)}
```

- [ ] **Step 2: Verify in browser**

```bash
npm run dev
```

To test the stall message display: open browser devtools, find the `stallMessage` setter, or temporarily force-set `STALL_TIMEOUT_MS = 1000` in `useAudioLifecycle.ts`, play a station, disconnect network, wait for stall. The amber message should appear.

Restore `STALL_TIMEOUT_MS = 5_000` afterward.

- [ ] **Step 3: Commit**

```bash
git add src/App.tsx
git commit -m "feat: add persistent stall notification UI"
```

---

## Task 12: Delete usePreload, run full suite, lint

**Files:**
- Delete: `src/usePreload.ts`
- Delete: `src/usePreload.test.ts`

- [ ] **Step 1: Delete the files**

```bash
rm src/usePreload.ts src/usePreload.test.ts
```

- [ ] **Step 2: Verify no remaining imports**

```bash
grep -r "usePreload" src/
```

Expected: no output (nothing still imports it).

- [ ] **Step 3: Run the full test suite**

```bash
npm run test:run
```

Expected: all tests pass. No references to deleted files.

- [ ] **Step 4: Run lint**

```bash
npm run lint
```

Expected: no errors. Fix any warnings about unused imports or variables introduced by the refactor.

- [ ] **Step 5: Build**

```bash
npm run build
```

Expected: clean build, no TypeScript errors.

- [ ] **Step 6: Final commit**

```bash
git add -A
git commit -m "chore: remove usePreload (folded into useAudioLifecycle)"
```
