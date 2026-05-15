# Swipe Gesture Recognition — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract swipe gesture handling into a `useSwipeGesture` hook with full guards, scroll prevention, pointer capture, and chevron flash feedback.

**Architecture:** A new `useSwipeGesture` hook mirrors the `useNavigation` pattern — it accepts action callbacks, tracks pointer state internally, and returns React event handler props to spread onto the full-screen container in App.tsx. App.tsx merges `swipeFlash` from the hook with `keyboardFlash` from `useNavigation` into a single `activeFlash` passed to `NavControls`. The existing inline pointer handlers in App.tsx are removed.

**Tech Stack:** React 19, TypeScript, Vitest + Testing Library

---

## Files

- **Create:** `src/useSwipeGesture.ts`
- **Create:** `src/useSwipeGesture.test.ts`
- **Modify:** `src/App.tsx` — remove `pointerStartRef`, `handlePointerDown`, `handlePointerUp`; wire up hook; merge flash states

---

### Task 1: Write failing tests for `useSwipeGesture`

**Files:**
- Create: `src/useSwipeGesture.test.ts`

- [ ] **Step 1: Create the test file**

```ts
import { renderHook, act } from '@testing-library/react'
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { useSwipeGesture } from './useSwipeGesture'
import type { Phase } from './types'

const mockSetPointerCapture = vi.fn()

const ptr = (x: number, y: number, overrides: Record<string, unknown> = {}): React.PointerEvent<Element> =>
  ({
    clientX: x,
    clientY: y,
    isPrimary: true,
    pointerId: 1,
    currentTarget: { setPointerCapture: mockSetPointerCapture },
    preventDefault: vi.fn(),
    ...overrides,
  } as unknown as React.PointerEvent<Element>)

const makeCallbacks = () => ({
  onNext: vi.fn(),
  onPrevious: vi.fn(),
  onSave: vi.fn(),
  onOpenShelf: vi.fn(),
})

const hook = (phase: Phase = 'playing', canGoPrevious = true) => {
  const cbs = makeCallbacks()
  const { result } = renderHook(() =>
    useSwipeGesture(phase, cbs.onNext, cbs.onPrevious, canGoPrevious, cbs.onSave, cbs.onOpenShelf)
  )
  return { result, cbs }
}

beforeEach(() => { mockSetPointerCapture.mockClear() })

describe('useSwipeGesture', () => {
  it('fires onNext for left swipe >= 50px', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(145, 305)))
    expect(cbs.onNext).toHaveBeenCalledOnce()
  })

  it('fires onPrevious for right swipe >= 50px', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(255, 305)))
    expect(cbs.onPrevious).toHaveBeenCalledOnce()
  })

  it('does not fire onPrevious when canGoPrevious is false', () => {
    const { result, cbs } = hook('playing', false)
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(255, 305)))
    expect(cbs.onPrevious).not.toHaveBeenCalled()
  })

  it('fires onSave for downward swipe >= 80px', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 200)))
    act(() => result.current.onPointerUp(ptr(205, 285)))
    expect(cbs.onSave).toHaveBeenCalledOnce()
  })

  it('fires onOpenShelf for upward swipe >= 80px', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(205, 215)))
    expect(cbs.onOpenShelf).toHaveBeenCalledOnce()
  })

  it('does not fire when horizontal swipe is below 50px', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(225, 305)))
    expect(cbs.onNext).not.toHaveBeenCalled()
    expect(cbs.onPrevious).not.toHaveBeenCalled()
  })

  it('does not fire when vertical swipe is below 80px', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(205, 365)))
    expect(cbs.onSave).not.toHaveBeenCalled()
  })

  it('ignores non-primary pointers', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300, { isPrimary: false })))
    act(() => result.current.onPointerUp(ptr(145, 305, { isPrimary: false })))
    expect(cbs.onNext).not.toHaveBeenCalled()
  })

  it('ignores pointerdown starting within left edge dead zone (< 20px)', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(10, 300)))
    act(() => result.current.onPointerUp(ptr(75, 305)))
    expect(cbs.onNext).not.toHaveBeenCalled()
    expect(cbs.onPrevious).not.toHaveBeenCalled()
  })

  it('does not fire when phase is not playing', () => {
    const { result, cbs } = hook('idle')
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(145, 305)))
    expect(cbs.onNext).not.toHaveBeenCalled()
  })

  it('calls setPointerCapture on pointerdown', () => {
    const { result } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    expect(mockSetPointerCapture).toHaveBeenCalledWith(1)
  })

  it('calls preventDefault on pointermove when vertical intent is clear', () => {
    const { result } = hook()
    const moveEvent = ptr(205, 215)
    act(() => result.current.onPointerDown(ptr(200, 200)))
    act(() => result.current.onPointerMove(moveEvent))
    expect(moveEvent.preventDefault).toHaveBeenCalled()
  })

  it('does not call preventDefault on pointermove for horizontal movement', () => {
    const { result } = hook()
    const moveEvent = ptr(215, 202)
    act(() => result.current.onPointerDown(ptr(200, 200)))
    act(() => result.current.onPointerMove(moveEvent))
    expect(moveEvent.preventDefault).not.toHaveBeenCalled()
  })

  it('cancels gesture on pointercancel — no action fires on subsequent pointerup', () => {
    const { result, cbs } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerCancel())
    act(() => result.current.onPointerUp(ptr(145, 305)))
    expect(cbs.onNext).not.toHaveBeenCalled()
  })

  it('returns swipeFlash direction matching the triggered action', () => {
    const { result } = hook()
    act(() => result.current.onPointerDown(ptr(200, 300)))
    act(() => result.current.onPointerUp(ptr(145, 305)))
    expect(result.current.swipeFlash).toEqual({ direction: 'right' })
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npm run test:run -- src/useSwipeGesture.test.ts
```

Expected: FAIL — "Cannot find module './useSwipeGesture'"

---

### Task 2: Implement `useSwipeGesture`

**Files:**
- Create: `src/useSwipeGesture.ts`

- [ ] **Step 1: Create the hook**

```ts
import { useEffect, useRef, useState } from 'react'
import type { Phase } from './types'
import type { FlashEvent } from './useNavigation'

const H_THRESHOLD = 50
const V_THRESHOLD = 80
const LEFT_DEAD_ZONE = 20
const VERTICAL_INTENT_PX = 10
const FLASH_DURATION_MS = 180

interface SwipeState {
  startX: number
  startY: number
  pointerId: number
}

export function useSwipeGesture(
  phase: Phase,
  onNext: () => void,
  onPrevious: () => void,
  canGoPrevious: boolean,
  onSave: () => void,
  onOpenShelf: () => void,
): {
  swipeFlash: FlashEvent | null
  onPointerDown: (e: React.PointerEvent<Element>) => void
  onPointerMove: (e: React.PointerEvent<Element>) => void
  onPointerUp: (e: React.PointerEvent<Element>) => void
  onPointerCancel: () => void
} {
  const swipeRef = useRef<SwipeState | null>(null)
  const [swipeFlash, setSwipeFlash] = useState<FlashEvent | null>(null)
  const flashTimeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  const phaseRef = useRef(phase)
  const onNextRef = useRef(onNext)
  const onPreviousRef = useRef(onPrevious)
  const canGoPreviousRef = useRef(canGoPrevious)
  const onSaveRef = useRef(onSave)
  const onOpenShelfRef = useRef(onOpenShelf)

  useEffect(() => { phaseRef.current = phase }, [phase])
  useEffect(() => { onNextRef.current = onNext }, [onNext])
  useEffect(() => { onPreviousRef.current = onPrevious }, [onPrevious])
  useEffect(() => { canGoPreviousRef.current = canGoPrevious }, [canGoPrevious])
  useEffect(() => { onSaveRef.current = onSave }, [onSave])
  useEffect(() => { onOpenShelfRef.current = onOpenShelf }, [onOpenShelf])

  const triggerFlash = (direction: FlashEvent['direction']) => {
    if (flashTimeoutRef.current) clearTimeout(flashTimeoutRef.current)
    setSwipeFlash({ direction })
    flashTimeoutRef.current = setTimeout(() => setSwipeFlash(null), FLASH_DURATION_MS)
  }

  const onPointerDown = (e: React.PointerEvent<Element>) => {
    if (!e.isPrimary) return
    if (e.clientX < LEFT_DEAD_ZONE) return
    if (phaseRef.current !== 'playing') return
    swipeRef.current = { startX: e.clientX, startY: e.clientY, pointerId: e.pointerId }
    e.currentTarget.setPointerCapture(e.pointerId)
  }

  const onPointerMove = (e: React.PointerEvent<Element>) => {
    if (!swipeRef.current || swipeRef.current.pointerId !== e.pointerId) return
    const dy = e.clientY - swipeRef.current.startY
    const dx = e.clientX - swipeRef.current.startX
    if (Math.abs(dy) > Math.abs(dx) && Math.abs(dy) > VERTICAL_INTENT_PX) {
      e.preventDefault()
    }
  }

  const onPointerUp = (e: React.PointerEvent<Element>) => {
    if (!swipeRef.current || swipeRef.current.pointerId !== e.pointerId) return
    const dx = e.clientX - swipeRef.current.startX
    const dy = e.clientY - swipeRef.current.startY
    swipeRef.current = null

    const absDx = Math.abs(dx)
    const absDy = Math.abs(dy)

    if (absDy > absDx && absDy >= V_THRESHOLD) {
      if (dy > 0) { onSaveRef.current(); triggerFlash('down') }
      else { onOpenShelfRef.current(); triggerFlash('up') }
    } else if (absDx >= H_THRESHOLD && absDx > absDy) {
      if (dx < 0) { onNextRef.current(); triggerFlash('right') }
      else if (canGoPreviousRef.current) { onPreviousRef.current(); triggerFlash('left') }
    }
  }

  const onPointerCancel = () => {
    swipeRef.current = null
  }

  return { swipeFlash, onPointerDown, onPointerMove, onPointerUp, onPointerCancel }
}
```

- [ ] **Step 2: Run tests to confirm they pass**

```bash
npm run test:run -- src/useSwipeGesture.test.ts
```

Expected: all tests PASS

- [ ] **Step 3: Commit**

```bash
git add src/useSwipeGesture.ts src/useSwipeGesture.test.ts
git commit -m "feat: add useSwipeGesture hook with guards, scroll prevention, and flash feedback"
```

---

### Task 3: Wire up in App.tsx

**Files:**
- Modify: `src/App.tsx`

- [ ] **Step 1: Remove `pointerStartRef` from the refs block**

Find and remove this line (~line 28):
```diff
- const pointerStartRef  = useRef<{ x: number; y: number } | null>(null)
```

- [ ] **Step 2: Add the import**

```diff
  import { useNavigation } from './useNavigation'
+ import { useSwipeGesture } from './useSwipeGesture'
```

- [ ] **Step 3: Remove the two inline handler functions**

Find and remove `handlePointerDown` and `handlePointerUp` (~lines 215–238):
```diff
- const handlePointerDown = (e: React.PointerEvent) => {
-   pointerStartRef.current = { x: e.clientX, y: e.clientY }
- }
-
- const handlePointerUp = (e: React.PointerEvent) => {
-   if (!pointerStartRef.current || phaseRef.current !== 'playing') {
-     pointerStartRef.current = null
-     return
-   }
-   const dx = e.clientX - pointerStartRef.current.x
-   const dy = e.clientY - pointerStartRef.current.y
-   pointerStartRef.current = null
-
-   const absDx = Math.abs(dx)
-   const absDy = Math.abs(dy)
-
-   if (absDy > absDx && absDy >= 80) {
-     if (dy > 0) handleSave()
-     else openShelf()
-   } else if (absDx >= 50 && absDx > absDy) {
-     if (dx < 0) goNext()
-     else goPrevious()
-   }
- }
```

- [ ] **Step 4: Add the hook call and merge flash states**

After the `useNavigation` call:
```diff
  const { keyboardFlash } = useNavigation(
    phase, goNext, goPrevious, canGoPrevious,
    handleSave, openShelf, closeShelf, shelfOpen, handlePlayPause,
  )
+
+ const { swipeFlash, onPointerDown, onPointerMove, onPointerUp, onPointerCancel } = useSwipeGesture(
+   phase, goNext, goPrevious, canGoPrevious, handleSave, openShelf,
+ )
+ const activeFlash = keyboardFlash ?? swipeFlash
```

- [ ] **Step 5: Update the full-screen div event handlers**

```diff
        <div
          style={{
            position: 'fixed', inset: 0,
            zIndex: 0,
            pointerEvents: phase === 'playing' ? 'auto' : 'none',
          }}
-         onPointerMove={() => station && reprimeIfNeeded(station)}
-         onPointerDown={e => { station && reprimeIfNeeded(station); handlePointerDown(e) }}
-         onPointerUp={handlePointerUp}
-         onPointerCancel={() => { pointerStartRef.current = null }}
+         onPointerMove={e => { station && reprimeIfNeeded(station); onPointerMove(e) }}
+         onPointerDown={e => { station && reprimeIfNeeded(station); onPointerDown(e) }}
+         onPointerUp={onPointerUp}
+         onPointerCancel={onPointerCancel}
        />
```

- [ ] **Step 6: Pass `activeFlash` to `NavControls`**

```diff
        <NavControls
          onNext={goNext}
          onPrevious={goPrevious}
          onSave={shelfOpen ? closeShelf : handleSave}
          onOpenShelf={openShelf}
          canGoPrevious={canGoPrevious}
-         keyboardFlash={keyboardFlash}
+         keyboardFlash={activeFlash}
        />
```

- [ ] **Step 7: Run the full test suite**

```bash
npm run test:run
```

Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add src/App.tsx
git commit -m "feat: wire useSwipeGesture into App, remove inline pointer handlers"
```
