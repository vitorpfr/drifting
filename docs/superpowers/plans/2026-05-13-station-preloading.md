# Station Pre-loading Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Load the next station's audio stream silently in the background while the current station is playing, so forward navigation is instant instead of showing a "connecting…" delay.

**Architecture:** A new `usePreload` hook manages a single background `AudioPlayer` that connects to the next predicted station immediately after playback starts. The pre-load expires after 30 seconds (to avoid wasting bandwidth for long listeners) and re-primes on the first pointer activity after expiry. `App.tsx` consumes the pre-loaded player on forward navigation instead of waiting for a fresh connection.

**Tech Stack:** React 19, TypeScript 5.7, Vitest + Testing Library

---

## File Map

| File | Change |
|---|---|
| `src/audioPlayer.ts` | Add `setVolume(v: number): void` method |
| `src/audioPlayer.test.ts` | Add one test for `setVolume` |
| `src/usePreload.ts` | New hook — owns pre-load state and lifecycle |
| `src/usePreload.test.ts` | New test file — full coverage of hook behaviours |
| `src/App.tsx` | Wire hook; update `loadAndPlay`; update overlay div |

---

## Task 1: Add `AudioPlayer.setVolume`

**Files:**
- Modify: `src/audioPlayer.ts`
- Modify: `src/audioPlayer.test.ts`

The pre-loaded player must buffer silently at volume 0. `AudioPlayer` has no way to set volume externally today — `fadeIn` always resets to 0, but `fadeOut(0)` calls `stop()`. We need a plain setter.

- [ ] **Step 1: Write the failing test**

  In `src/audioPlayer.test.ts`, add inside the `describe('AudioPlayer', ...)` block (after the existing `stop() with fades` describe block, before the closing `}`):

  ```typescript
  describe('setVolume', () => {
    it('sets the audio element volume directly', () => {
      const player = new AudioPlayer()
      player.setVolume(0)
      expect(mockAudioElement.volume).toBe(0)
      player.setVolume(0.5)
      expect(mockAudioElement.volume).toBeCloseTo(0.5)
    })
  })
  ```

- [ ] **Step 2: Run test to verify it fails**

  ```bash
  npm run test:run -- --reporter=verbose 2>&1 | grep -A3 "setVolume"
  ```

  Expected: `TypeError: player.setVolume is not a function`

- [ ] **Step 3: Implement `setVolume`**

  In `src/audioPlayer.ts`, add after the `fadeIn` method (before the closing `}`):

  ```typescript
  setVolume(v: number): void {
    this.audio.volume = v
  }
  ```

- [ ] **Step 4: Run tests to verify pass**

  ```bash
  npm run test:run -- --reporter=verbose 2>&1 | grep -A3 "setVolume"
  ```

  Expected: `✓ sets the audio element volume directly`

- [ ] **Step 5: Commit**

  ```bash
  git add src/audioPlayer.ts src/audioPlayer.test.ts
  git commit -m "feat: add AudioPlayer.setVolume for silent pre-loading"
  ```

---

## Task 2: Write `usePreload` tests

**Files:**
- Create: `src/usePreload.test.ts`

Write all tests before any implementation. Every test will fail because the module doesn't exist yet — that's expected.

- [ ] **Step 1: Create the test file**

  Create `src/usePreload.test.ts` with the full contents below:

  ```typescript
  import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
  import { renderHook, act } from '@testing-library/react'
  import type { Station } from './types'

  const mockPlay = vi.fn()
  const mockStop = vi.fn()
  const mockSetVolume = vi.fn()

  vi.mock('./audioPlayer', () => ({
    AudioPlayer: vi.fn(() => ({ play: mockPlay, stop: mockStop, setVolume: mockSetVolume })),
  }))

  import { usePreload } from './usePreload'

  const currentStation: Station = {
    stationuuid: 'current-1',
    name: 'Current FM',
    url: 'http://current.example.com/stream',
    tags: 'jazz',
    country: 'France',
    bitrate: 128,
    corsFriendly: false,
  }

  const nextStation: Station = {
    stationuuid: 'next-1',
    name: 'Next FM',
    url: 'http://next.example.com/stream',
    tags: 'rock',
    country: 'Germany',
    bitrate: 128,
    corsFriendly: false,
  }

  const catalog = [currentStation, nextStation]

  function setup(catalogValue: Station[] | null = catalog) {
    const getCatalog = () => catalogValue
    const getSavedStations = () => []
    const selectNext = vi.fn().mockReturnValue(nextStation)
    const hook = renderHook(() => usePreload(getCatalog, getSavedStations, selectNext))
    return { ...hook, selectNext }
  }

  beforeEach(() => {
    vi.clearAllMocks()
    mockPlay.mockResolvedValue(undefined)
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  describe('usePreload', () => {
    describe('prime', () => {
      it('calls selectNext and starts play on the returned station', async () => {
        const { result, selectNext } = setup()

        await act(async () => { result.current.prime(currentStation) })

        expect(selectNext).toHaveBeenCalledWith(catalog, [], currentStation)
        expect(mockSetVolume).toHaveBeenCalledWith(0)
        expect(mockPlay).toHaveBeenCalledWith(nextStation.url, false)
      })

      it('does nothing when catalog is null', () => {
        const { result } = setup(null)

        act(() => { result.current.prime(currentStation) })

        expect(mockPlay).not.toHaveBeenCalled()
      })

      it('marks pre-load as ready once play resolves', async () => {
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })

        expect(result.current.consume()).not.toBeNull()
      })

      it('leaves pre-load not ready while play has not resolved', () => {
        mockPlay.mockImplementation(() => new Promise(() => {}))
        const { result } = setup()

        act(() => { result.current.prime(currentStation) })

        expect(result.current.consume()).toBeNull()
      })

      it('silently clears pre-load when play rejects', async () => {
        mockPlay.mockRejectedValue(new Error('network error'))
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })

        expect(result.current.consume()).toBeNull()
        expect(mockStop).toHaveBeenCalledTimes(1)
      })

      it('stops the existing pre-load before starting a new one', async () => {
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })
        expect(mockStop).not.toHaveBeenCalled()

        await act(async () => { result.current.prime(currentStation) })
        expect(mockStop).toHaveBeenCalledTimes(1)
      })
    })

    describe('consume', () => {
      it('returns { player, station } when pre-load is ready', async () => {
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })
        const result_ = result.current.consume()

        expect(result_).not.toBeNull()
        expect(result_?.station).toEqual(nextStation)
      })

      it('returns null when pre-load is not ready', () => {
        mockPlay.mockImplementation(() => new Promise(() => {}))
        const { result } = setup()

        act(() => { result.current.prime(currentStation) })

        expect(result.current.consume()).toBeNull()
      })

      it('returns null when nothing has been pre-loaded', () => {
        const { result } = setup()
        expect(result.current.consume()).toBeNull()
      })

      it('clears the ref so a second consume returns null', async () => {
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })

        const first = result.current.consume()
        const second = result.current.consume()

        expect(first).not.toBeNull()
        expect(second).toBeNull()
      })
    })

    describe('expiry', () => {
      it('stops player and clears pre-load after 30 seconds', async () => {
        vi.useFakeTimers()
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })
        expect(result.current.consume()).not.toBeNull()

        // Re-prime to get a fresh pre-load for expiry test
        await act(async () => { result.current.prime(currentStation) })
        act(() => { vi.advanceTimersByTime(30_000) })

        expect(result.current.consume()).toBeNull()
        expect(mockStop).toHaveBeenCalled()
      })

      it('does not expire before 30 seconds', async () => {
        vi.useFakeTimers()
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })
        act(() => { vi.advanceTimersByTime(29_999) })

        expect(result.current.consume()).not.toBeNull()
      })
    })

    describe('reprimeIfNeeded', () => {
      it('primes when nothing is pre-loaded', async () => {
        const { result } = setup()

        await act(async () => { result.current.reprimeIfNeeded(currentStation) })

        expect(result.current.consume()).not.toBeNull()
      })

      it('is a no-op when a pre-load is already active', async () => {
        const { result } = setup()

        await act(async () => { result.current.prime(currentStation) })
        vi.clearAllMocks()

        await act(async () => { result.current.reprimeIfNeeded(currentStation) })

        expect(mockPlay).not.toHaveBeenCalled()
        expect(mockStop).not.toHaveBeenCalled()
      })
    })

    describe('cleanup', () => {
      it('stops the active pre-load on unmount', async () => {
        const { result, unmount } = setup()

        await act(async () => { result.current.prime(currentStation) })
        expect(mockStop).not.toHaveBeenCalled()

        unmount()

        expect(mockStop).toHaveBeenCalledTimes(1)
      })
    })
  })
  ```

- [ ] **Step 2: Run tests to verify they all fail (module not found)**

  ```bash
  npm run test:run -- --reporter=verbose src/usePreload.test.ts 2>&1 | tail -20
  ```

  Expected: `Error: Failed to resolve import "./usePreload"` (or similar — all tests fail because the file doesn't exist)

---

## Task 3: Implement `usePreload`

**Files:**
- Create: `src/usePreload.ts`

- [ ] **Step 1: Create the implementation**

  Create `src/usePreload.ts`:

  ```typescript
  import { useCallback, useEffect, useRef } from 'react'
  import { AudioPlayer } from './audioPlayer'
  import type { Station } from './types'

  export function usePreload(
    getCatalog: () => Station[] | null,
    getSavedStations: () => Station[],
    selectNext: (catalog: Station[], saved: Station[], last: Station | null) => Station,
  ): {
    prime: (currentStation: Station) => void
    consume: () => { player: AudioPlayer; station: Station } | null
    reprimeIfNeeded: (currentStation: Station) => void
  } {
    const preloadRef = useRef<{
      player: AudioPlayer
      station: Station
      ready: boolean
      expiryTimer: ReturnType<typeof setTimeout>
    } | null>(null)

    const cancelPreload = useCallback(() => {
      if (preloadRef.current) {
        clearTimeout(preloadRef.current.expiryTimer)
        preloadRef.current.player.stop()
        preloadRef.current = null
      }
    }, [])

    const prime = useCallback((currentStation: Station) => {
      cancelPreload()
      const catalog = getCatalog()
      if (!catalog) return

      const nextStation = selectNext(catalog, getSavedStations(), currentStation)
      const player = new AudioPlayer()
      player.setVolume(0)

      const expiryTimer = setTimeout(() => {
        if (preloadRef.current?.expiryTimer === expiryTimer) {
          preloadRef.current.player.stop()
          preloadRef.current = null
        }
      }, 30_000)

      preloadRef.current = { player, station: nextStation, ready: false, expiryTimer }

      player.play(nextStation.url, false).then(() => {
        if (preloadRef.current?.player === player) {
          preloadRef.current.ready = true
        }
      }).catch(() => {
        if (preloadRef.current?.player === player) {
          clearTimeout(preloadRef.current.expiryTimer)
          preloadRef.current.player.stop()
          preloadRef.current = null
        }
      })
    }, [cancelPreload, getCatalog, getSavedStations, selectNext])

    const consume = useCallback((): { player: AudioPlayer; station: Station } | null => {
      if (!preloadRef.current?.ready) return null
      const { player, station, expiryTimer } = preloadRef.current
      clearTimeout(expiryTimer)
      preloadRef.current = null
      return { player, station }
    }, [])

    const reprimeIfNeeded = useCallback((currentStation: Station) => {
      if (!preloadRef.current) prime(currentStation)
    }, [prime])

    useEffect(() => () => cancelPreload(), [cancelPreload])

    return { prime, consume, reprimeIfNeeded }
  }
  ```

- [ ] **Step 2: Run usePreload tests to verify they all pass**

  ```bash
  npm run test:run -- --reporter=verbose src/usePreload.test.ts 2>&1 | tail -30
  ```

  Expected: all tests green, 0 failures.

- [ ] **Step 3: Run the full suite to verify no regressions**

  ```bash
  npm run test:run 2>&1 | tail -10
  ```

  Expected: all existing tests still pass.

- [ ] **Step 4: Commit**

  ```bash
  git add src/usePreload.ts src/usePreload.test.ts
  git commit -m "feat: add usePreload hook for background station loading"
  ```

---

## Task 4: Wire `App.tsx` — `loadAndPlay` + `prime`

**Files:**
- Modify: `src/App.tsx`

This task updates `loadAndPlay` to use the pre-loaded player on forward navigation, and calls `prime` after each station starts.

- [ ] **Step 1: Add import and hook instantiation**

  In `src/App.tsx`, add `usePreload` to the imports line (after the `useNowPlaying` import):

  ```typescript
  import { useNowPlaying } from './useNowPlaying'
  import { usePreload } from './usePreload'
  ```

  Then add the hook call after the `nowPlaying` line (around line 43):

  ```typescript
  const nowPlaying = useNowPlaying(station)
  const { prime, consume, reprimeIfNeeded } = usePreload(
    () => catalogRef.current,
    () => savedStationsRef.current,
    pickNext,
  )
  ```

- [ ] **Step 2: Update the initial-load path in `loadAndPlay`**

  The initial path ends with `return true` around line 81. Add `prime(s)` just before `setPhase('exiting')`:

  Replace:
  ```typescript
          setStation(s)
          setPhase('exiting')
          setTimeout(() => { setPhase('playing'); setIsFirstLoad(false) }, 800)
          return true
  ```

  With:
  ```typescript
          setStation(s)
          prime(s)
          setPhase('exiting')
          setTimeout(() => { setPhase('playing'); setIsFirstLoad(false) }, 800)
          return true
  ```

- [ ] **Step 3: Update the navigation path in `loadAndPlay`**

  The navigation path (non-initial) currently reads:

  ```typescript
      const s = stationToLoad ?? pickNext(catalogRef.current, savedStationsRef.current, stationRef.current)
      const player = new AudioPlayer()
      await player.play(s.url, false)   // new stream ready; old still playing at full volume
      oldPlayer?.fadeOut(400)           // crossfade now — no silence gap
      playerRef.current = player
      setPaused(false)
      vizRef.current?.loadStation({ ...s, corsFriendly: false }, player)
      vizRef.current?.triggerBurst()
      player.fadeIn(1000)
      playStartRef.current = Date.now()
      startDwellTimer(s)
      setStation(s)
      return true
  ```

  Replace with:

  ```typescript
      const preloaded = !stationToLoad ? consume() : null
      const s = stationToLoad ?? preloaded?.station ?? pickNext(catalogRef.current, savedStationsRef.current, stationRef.current)
      const player = preloaded?.player ?? new AudioPlayer()
      if (!preloaded) await player.play(s.url, false)
      oldPlayer?.fadeOut(400)
      playerRef.current = player
      setPaused(false)
      vizRef.current?.loadStation({ ...s, corsFriendly: false }, player)
      vizRef.current?.triggerBurst()
      player.fadeIn(1000)
      playStartRef.current = Date.now()
      startDwellTimer(s)
      setStation(s)
      prime(s)
      return true
  ```

- [ ] **Step 4: Add `prime` and `consume` to `loadAndPlay` dependency array**

  The `useCallback` dependency array for `loadAndPlay` currently reads:

  ```typescript
  }, [pickNext, startDwellTimer])
  ```

  Replace with:

  ```typescript
  }, [pickNext, startDwellTimer, prime, consume])
  ```

- [ ] **Step 5: Run the full test suite**

  ```bash
  npm run test:run 2>&1 | tail -10
  ```

  Expected: all tests pass.

- [ ] **Step 6: Commit**

  ```bash
  git add src/App.tsx
  git commit -m "feat: use pre-loaded player in loadAndPlay for instant navigation"
  ```

---

## Task 5: Wire `App.tsx` — pointer handlers for re-prime

**Files:**
- Modify: `src/App.tsx`

After the 30-second expiry drops the pre-load, the first pointer activity should re-prime it.

- [ ] **Step 1: Update the overlay div**

  Find the overlay div (the one with `onPointerDown={handlePointerDown}`). It currently reads:

  ```tsx
        <div
          style={{
            position: 'fixed', inset: 0,
            zIndex: 0,
            pointerEvents: phase === 'playing' ? 'auto' : 'none',
          }}
          onPointerDown={handlePointerDown}
          onPointerUp={handlePointerUp}
          onPointerCancel={() => { pointerStartRef.current = null }}
        />
  ```

  Replace with:

  ```tsx
        <div
          style={{
            position: 'fixed', inset: 0,
            zIndex: 0,
            pointerEvents: phase === 'playing' ? 'auto' : 'none',
          }}
          onPointerMove={() => station && reprimeIfNeeded(station)}
          onPointerDown={e => { station && reprimeIfNeeded(station); handlePointerDown(e) }}
          onPointerUp={handlePointerUp}
          onPointerCancel={() => { pointerStartRef.current = null }}
        />
  ```

  `onPointerMove` covers mouse movement and touch drag. `onPointerDown` covers tap-without-move. `reprimeIfNeeded` is a ref check — safe to call on every pointer event.

- [ ] **Step 2: Run the full test suite**

  ```bash
  npm run test:run 2>&1 | tail -10
  ```

  Expected: all tests pass.

- [ ] **Step 3: Run the dev server and manually verify**

  ```bash
  npm run dev
  ```

  1. Open the app, tap to begin, wait for a station to play.
  2. Immediately swipe/press → next. The transition should be noticeably faster (no "connecting…" delay if the pre-load finished).
  3. Let a station play for >30 seconds. Move the mouse or touch the screen — re-prime starts.
  4. Swipe next — should still transition smoothly (pre-load had time to connect after the pointer activity).

- [ ] **Step 4: Final commit**

  ```bash
  git add src/App.tsx
  git commit -m "feat: re-prime pre-load on pointer activity after expiry"
  ```
