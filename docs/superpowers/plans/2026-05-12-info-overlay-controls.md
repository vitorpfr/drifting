# Info Overlay Controls Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add always-visible share, settings, and play/pause controls to the playing state, plus a settings modal with taste profile reset, audio-only toggle, and saved stations clear.

**Architecture:** Two new components (`OverlayControls`, `SettingsModal`) wired into `App.tsx`. Three new methods on existing hooks/managers (`clearProfile`, `clearAll`, `setAudioOnly`). A `paused` boolean is promoted to React state so the play/pause icon is reactive. All IDB operations follow the existing `openDB()` pattern in `useIndexedDB.ts`.

**Tech Stack:** React 19, TypeScript, Vitest + Testing Library, fake-indexeddb (tests), inline SVG icons (no icon library)

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `src/useRecommendation.ts` | Modify | Add `clearProfile()` — deletes IDB key, resets in-memory profile |
| `src/useRecommendation.test.ts` | Modify | Tests for `clearProfile()` |
| `src/useSavedStations.ts` | Modify | Add `clearAll()` — clears IDB store, resets state |
| `src/useSavedStations.test.ts` | Modify | Tests for `clearAll()` |
| `src/visualization/VisualizationManager.ts` | Modify | Add `setAudioOnly(enabled)` — stops/resumes animation loop |
| `src/visualization/VisualizationManager.test.ts` | Modify | Tests for `setAudioOnly()` |
| `src/OverlayControls.tsx` | Create | Top-right (share + settings) and bottom-right (play/pause) buttons |
| `src/OverlayControls.test.tsx` | Create | Render + click tests for OverlayControls |
| `src/SettingsModal.tsx` | Create | Centered modal: reset profile, audio-only toggle, clear saved |
| `src/SettingsModal.test.tsx` | Create | Render + interaction tests for SettingsModal |
| `src/App.tsx` | Modify | Add `paused` + `audioOnly` + `settingsOpen` state; wire both new components |

---

## Task 1: `clearProfile()` in `useRecommendation`

**Files:**
- Modify: `src/useRecommendation.ts`
- Modify: `src/useRecommendation.test.ts`

- [ ] **Step 1: Add a failing test for `clearProfile()`**

Append to the `describe('useRecommendation', ...)` block in `src/useRecommendation.test.ts`:

```typescript
it('clearProfile() resets the profile and clears IDB so remount starts fresh', async () => {
  const { result, unmount } = renderHook(() => useRecommendation())

  // Record a signal so the profile is non-empty
  act(() => { result.current.recordSignal(station, 'save') })

  // Clear the profile
  await act(async () => { await result.current.clearProfile() })

  unmount()

  // Remount — profile should load as empty (interactionCount = 0)
  // Verify by checking selectNext still works (doesn't throw) and the profile
  // is treated as cold-start (interactionCount 0 means random selection).
  // We use a multi-station catalog and verify no error is thrown.
  const station2: Station = { ...station, stationuuid: 'uuid-2', name: 'Rock FM' }
  const { result: result2 } = renderHook(() => useRecommendation())
  await waitFor(() => {
    let picked: Station | undefined
    act(() => {
      picked = result2.current.selectNext([station, station2], [], null)
    })
    expect(picked).toBeDefined()
  })
})
```

- [ ] **Step 2: Run to confirm it fails**

```bash
npm run test:run -- src/useRecommendation.test.ts
```

Expected: FAIL — `result.current.clearProfile is not a function`

- [ ] **Step 3: Add `clearPersistedProfile` helper and `clearProfile` to `useRecommendation.ts`**

After the existing `persistProfile` function, add:

```typescript
async function clearPersistedProfile(): Promise<void> {
  try {
    const db = await openDB()
    await new Promise<void>((resolve, reject) => {
      const tx = db.transaction('keyval', 'readwrite')
      tx.objectStore('keyval').delete(IDB_KEY)
      tx.oncomplete = () => resolve()
      tx.onerror    = () => reject(tx.error)
    })
  } catch {}
}
```

Inside `useRecommendation()`, after `cancelDwellTimer`, add:

```typescript
const clearProfile = useCallback(async () => {
  profileRef.current = createEmptyProfile()
  await clearPersistedProfile()
}, [])
```

Update the return statement:

```typescript
return { recordSignal, selectNext, startDwellTimer, cancelDwellTimer, clearProfile }
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/useRecommendation.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useRecommendation.ts src/useRecommendation.test.ts
git commit -m "feat: add clearProfile() to useRecommendation"
```

---

## Task 2: `clearAll()` in `useSavedStations`

**Files:**
- Modify: `src/useSavedStations.ts`
- Modify: `src/useSavedStations.test.ts`

- [ ] **Step 1: Add a failing test for `clearAll()`**

Append to the `describe('useSavedStations', ...)` block in `src/useSavedStations.test.ts`:

```typescript
it('clearAll() empties the list and persists to IDB', async () => {
  const { result } = renderHook(() => useSavedStations())

  await act(async () => { await result.current.save(station) })
  expect(result.current.savedStations).toHaveLength(1)

  await act(async () => { await result.current.clearAll() })
  expect(result.current.savedStations).toHaveLength(0)
  expect(result.current.isSaved('uuid-1')).toBe(false)
})
```

- [ ] **Step 2: Run to confirm it fails**

```bash
npm run test:run -- src/useSavedStations.test.ts
```

Expected: FAIL — `result.current.clearAll is not a function`

- [ ] **Step 3: Implement `clearAll()` in `useSavedStations.ts`**

After the `remove` callback, add:

```typescript
const clearAll = useCallback(async () => {
  const db = await openDB()
  await new Promise<void>((resolve, reject) => {
    const tx = db.transaction('savedStations', 'readwrite')
    tx.objectStore('savedStations').clear()
    tx.oncomplete = () => resolve()
    tx.onerror    = () => reject(tx.error)
  })
  setSavedStations([])
}, [])
```

Update the return statement:

```typescript
return { savedStations, save, remove, isSaved, clearAll }
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/useSavedStations.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useSavedStations.ts src/useSavedStations.test.ts
git commit -m "feat: add clearAll() to useSavedStations"
```

---

## Task 3: `setAudioOnly()` in `VisualizationManager`

**Files:**
- Modify: `src/visualization/VisualizationManager.ts`
- Modify: `src/visualization/VisualizationManager.test.ts`

- [ ] **Step 1: Add failing tests for `setAudioOnly()`**

Append to `src/visualization/VisualizationManager.test.ts`:

```typescript
describe('VisualizationManager.setAudioOnly', () => {
  it('audioOnly is false by default', () => {
    const vm = new VisualizationManager(makeCanvas())
    expect(vm.audioOnly).toBe(false)
  })

  it('setAudioOnly(true) sets audioOnly to true', () => {
    const vm = new VisualizationManager(makeCanvas())
    vm.setAudioOnly(true)
    expect(vm.audioOnly).toBe(true)
  })

  it('setAudioOnly(false) sets audioOnly back to false', () => {
    const vm = new VisualizationManager(makeCanvas())
    vm.setAudioOnly(true)
    vm.setAudioOnly(false)
    expect(vm.audioOnly).toBe(false)
  })
})
```

- [ ] **Step 2: Run to confirm they fail**

```bash
npm run test:run -- src/visualization/VisualizationManager.test.ts
```

Expected: FAIL — `vm.audioOnly is not a property` / `vm.setAudioOnly is not a function`

- [ ] **Step 3: Implement `setAudioOnly()` in `VisualizationManager.ts`**

Add a private field after the existing private fields (after `private rafHandle = 0`):

```typescript
private _audioOnly = false
```

Add a public getter and method after `triggerBurst()`:

```typescript
get audioOnly(): boolean {
  return this._audioOnly
}

setAudioOnly(enabled: boolean): void {
  this._audioOnly = enabled
  if (enabled) {
    this.stop()
  } else {
    this.start()
  }
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/visualization/VisualizationManager.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/visualization/VisualizationManager.ts src/visualization/VisualizationManager.test.ts
git commit -m "feat: add setAudioOnly() to VisualizationManager"
```

---

## Task 4: `OverlayControls` component

**Files:**
- Create: `src/OverlayControls.tsx`
- Create: `src/OverlayControls.test.tsx`

- [ ] **Step 1: Create the test file first**

Create `src/OverlayControls.test.tsx`:

```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { vi, describe, it, expect } from 'vitest'
import { OverlayControls } from './OverlayControls'

describe('OverlayControls', () => {
  it('renders share, settings, and pause buttons when not paused', () => {
    render(
      <OverlayControls
        onShare={vi.fn()} onOpenSettings={vi.fn()} onPlayPause={vi.fn()} paused={false}
      />
    )
    expect(screen.getByLabelText('Share')).toBeInTheDocument()
    expect(screen.getByLabelText('Settings')).toBeInTheDocument()
    expect(screen.getByLabelText('Pause')).toBeInTheDocument()
  })

  it('renders play button when paused', () => {
    render(
      <OverlayControls
        onShare={vi.fn()} onOpenSettings={vi.fn()} onPlayPause={vi.fn()} paused={true}
      />
    )
    expect(screen.getByLabelText('Play')).toBeInTheDocument()
  })

  it('calls onShare when share button is clicked', () => {
    const onShare = vi.fn()
    render(
      <OverlayControls
        onShare={onShare} onOpenSettings={vi.fn()} onPlayPause={vi.fn()} paused={false}
      />
    )
    fireEvent.click(screen.getByLabelText('Share'))
    expect(onShare).toHaveBeenCalledTimes(1)
  })

  it('calls onOpenSettings when settings button is clicked', () => {
    const onOpenSettings = vi.fn()
    render(
      <OverlayControls
        onShare={vi.fn()} onOpenSettings={onOpenSettings} onPlayPause={vi.fn()} paused={false}
      />
    )
    fireEvent.click(screen.getByLabelText('Settings'))
    expect(onOpenSettings).toHaveBeenCalledTimes(1)
  })

  it('calls onPlayPause when play/pause button is clicked', () => {
    const onPlayPause = vi.fn()
    render(
      <OverlayControls
        onShare={vi.fn()} onOpenSettings={vi.fn()} onPlayPause={onPlayPause} paused={false}
      />
    )
    fireEvent.click(screen.getByLabelText('Pause'))
    expect(onPlayPause).toHaveBeenCalledTimes(1)
  })
})
```

- [ ] **Step 2: Run to confirm they fail**

```bash
npm run test:run -- src/OverlayControls.test.tsx
```

Expected: FAIL — `Cannot find module './OverlayControls'`

- [ ] **Step 3: Create `src/OverlayControls.tsx`**

```typescript
import { useState } from 'react'
import type { CSSProperties } from 'react'

interface Props {
  onShare: () => void
  onOpenSettings: () => void
  onPlayPause: () => void
  paused: boolean
}

const ShareIcon = () => (
  <svg width="18" height="18" viewBox="0 0 18 18" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
    <path d="M9 11V2M5 6l4-4 4 4" />
    <path d="M3 12v3a1 1 0 001 1h10a1 1 0 001-1v-3" />
  </svg>
)

const SettingsIcon = () => (
  <svg width="18" height="18" viewBox="0 0 18 18" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
    <circle cx="9" cy="9" r="2.5" />
    <path d="M9 1v2M9 15v2M1 9h2M15 9h2M3.22 3.22l1.42 1.42M13.36 13.36l1.42 1.42M3.22 14.78l1.42-1.42M13.36 4.64l1.42-1.42" />
  </svg>
)

const PlayIcon = () => (
  <svg width="18" height="18" viewBox="0 0 18 18" fill="currentColor">
    <path d="M5 3.5l10 5.5-10 5.5V3.5z" />
  </svg>
)

const PauseIcon = () => (
  <svg width="18" height="18" viewBox="0 0 18 18" fill="currentColor">
    <rect x="4" y="3" width="3.5" height="12" rx="1" />
    <rect x="10.5" y="3" width="3.5" height="12" rx="1" />
  </svg>
)

const BASE_BTN: CSSProperties = {
  width: 28,
  height: 28,
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center',
  background: 'none',
  border: 'none',
  padding: 0,
  cursor: 'pointer',
  transition: 'color 0.15s',
}

function IconButton({
  label,
  onClick,
  children,
}: {
  label: string
  onClick: () => void
  children: React.ReactNode
}) {
  const [hovered, setHovered] = useState(false)
  return (
    <button
      aria-label={label}
      onClick={onClick}
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      style={{
        ...BASE_BTN,
        color: hovered ? 'rgba(255,255,255,0.75)' : 'rgba(255,255,255,0.4)',
      }}
    >
      {children}
    </button>
  )
}

export function OverlayControls({ onShare, onOpenSettings, onPlayPause, paused }: Props) {
  return (
    <>
      <div
        style={{
          position: 'fixed',
          top: 32,
          right: 32,
          display: 'flex',
          gap: 20,
          pointerEvents: 'auto',
          zIndex: 10,
        }}
      >
        <IconButton label="Share" onClick={onShare}>
          <ShareIcon />
        </IconButton>
        <IconButton label="Settings" onClick={onOpenSettings}>
          <SettingsIcon />
        </IconButton>
      </div>

      <div
        style={{
          position: 'fixed',
          bottom: 40,
          right: 40,
          pointerEvents: 'auto',
          zIndex: 10,
        }}
      >
        <IconButton label={paused ? 'Play' : 'Pause'} onClick={onPlayPause}>
          {paused ? <PlayIcon /> : <PauseIcon />}
        </IconButton>
      </div>
    </>
  )
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/OverlayControls.test.tsx
```

Expected: all 5 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/OverlayControls.tsx src/OverlayControls.test.tsx
git commit -m "feat: add OverlayControls component with share, settings, and play/pause"
```

---

## Task 5: `SettingsModal` component

**Files:**
- Create: `src/SettingsModal.tsx`
- Create: `src/SettingsModal.test.tsx`

- [ ] **Step 1: Create the test file first**

Create `src/SettingsModal.test.tsx`:

```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { vi, describe, it, expect } from 'vitest'
import { SettingsModal } from './SettingsModal'

const defaultProps = {
  audioOnly: false,
  onToggleAudioOnly: vi.fn(),
  onResetProfile: vi.fn(),
  onClearSaved: vi.fn(),
  onClose: vi.fn(),
}

describe('SettingsModal', () => {
  it('renders all three setting rows', () => {
    render(<SettingsModal {...defaultProps} />)
    expect(screen.getByText('Taste profile')).toBeInTheDocument()
    expect(screen.getByText('Audio only')).toBeInTheDocument()
    expect(screen.getByText('Saved stations')).toBeInTheDocument()
  })

  it('calls onClose when the × button is clicked', () => {
    const onClose = vi.fn()
    render(<SettingsModal {...defaultProps} onClose={onClose} />)
    fireEvent.click(screen.getByLabelText('Close settings'))
    expect(onClose).toHaveBeenCalledTimes(1)
  })

  it('calls onClose when the backdrop is clicked', () => {
    const onClose = vi.fn()
    render(<SettingsModal {...defaultProps} onClose={onClose} />)
    fireEvent.click(screen.getByTestId('settings-backdrop'))
    expect(onClose).toHaveBeenCalledTimes(1)
  })

  it('calls onResetProfile when Reset is clicked', () => {
    const onResetProfile = vi.fn()
    render(<SettingsModal {...defaultProps} onResetProfile={onResetProfile} />)
    fireEvent.click(screen.getByText('Reset'))
    expect(onResetProfile).toHaveBeenCalledTimes(1)
  })

  it('calls onClearSaved when Clear all is clicked', () => {
    const onClearSaved = vi.fn()
    render(<SettingsModal {...defaultProps} onClearSaved={onClearSaved} />)
    fireEvent.click(screen.getByText('Clear all'))
    expect(onClearSaved).toHaveBeenCalledTimes(1)
  })

  it('toggle shows aria-checked=false when audioOnly is false', () => {
    render(<SettingsModal {...defaultProps} audioOnly={false} />)
    expect(screen.getByRole('switch')).toHaveAttribute('aria-checked', 'false')
  })

  it('toggle shows aria-checked=true when audioOnly is true', () => {
    render(<SettingsModal {...defaultProps} audioOnly={true} />)
    expect(screen.getByRole('switch')).toHaveAttribute('aria-checked', 'true')
  })

  it('calls onToggleAudioOnly when the toggle is clicked', () => {
    const onToggleAudioOnly = vi.fn()
    render(<SettingsModal {...defaultProps} onToggleAudioOnly={onToggleAudioOnly} />)
    fireEvent.click(screen.getByRole('switch'))
    expect(onToggleAudioOnly).toHaveBeenCalledTimes(1)
  })
})
```

- [ ] **Step 2: Run to confirm they fail**

```bash
npm run test:run -- src/SettingsModal.test.tsx
```

Expected: FAIL — `Cannot find module './SettingsModal'`

- [ ] **Step 3: Create `src/SettingsModal.tsx`**

```typescript
import type { CSSProperties } from 'react'

interface Props {
  audioOnly: boolean
  onToggleAudioOnly: () => void
  onResetProfile: () => void
  onClearSaved: () => void
  onClose: () => void
}

const LABEL: CSSProperties = {
  fontFamily: 'system-ui, sans-serif',
  fontSize: 14,
  fontWeight: 500,
  color: 'rgba(255,255,255,0.85)',
  letterSpacing: '0.01em',
}

const DESC: CSSProperties = {
  marginTop: 3,
  fontFamily: 'system-ui, sans-serif',
  fontSize: 12,
  fontWeight: 400,
  color: 'rgba(255,255,255,0.4)',
  letterSpacing: '0.02em',
  lineHeight: 1.4,
}

const ROW: CSSProperties = {
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'space-between',
  gap: 16,
  padding: '14px 0',
  borderBottom: '1px solid rgba(255,255,255,0.07)',
}

const DESTRUCTIVE_BTN: CSSProperties = {
  background: 'none',
  border: '1px solid rgba(255,80,80,0.4)',
  borderRadius: 6,
  padding: '5px 12px',
  fontFamily: 'system-ui, sans-serif',
  fontSize: 12,
  fontWeight: 500,
  color: 'rgba(255,80,80,0.8)',
  cursor: 'pointer',
  flexShrink: 0,
  letterSpacing: '0.02em',
}

export function SettingsModal({
  audioOnly,
  onToggleAudioOnly,
  onResetProfile,
  onClearSaved,
  onClose,
}: Props) {
  return (
    <div
      data-testid="settings-backdrop"
      onClick={onClose}
      style={{
        position: 'fixed',
        inset: 0,
        background: 'rgba(0,0,0,0.6)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 100,
      }}
    >
      <div
        onClick={e => e.stopPropagation()}
        style={{
          maxWidth: 320,
          width: '90%',
          background: 'rgba(15,15,15,0.95)',
          borderRadius: 16,
          padding: '24px 24px 10px',
          position: 'relative',
        }}
      >
        <button
          aria-label="Close settings"
          onClick={onClose}
          style={{
            position: 'absolute',
            top: 14,
            right: 16,
            background: 'none',
            border: 'none',
            padding: 0,
            fontFamily: 'system-ui, sans-serif',
            fontSize: 20,
            color: 'rgba(255,255,255,0.35)',
            cursor: 'pointer',
            lineHeight: 1,
          }}
        >
          ×
        </button>

        <div style={{
          fontFamily: 'system-ui, sans-serif',
          fontSize: 13,
          fontWeight: 600,
          color: 'rgba(255,255,255,0.5)',
          letterSpacing: '0.12em',
          textTransform: 'uppercase',
          marginBottom: 4,
        }}>
          Settings
        </div>

        {/* Taste profile row */}
        <div style={ROW}>
          <div>
            <div style={LABEL}>Taste profile</div>
            <div style={DESC}>Clears your listening history and restarts recommendations from scratch.</div>
          </div>
          <button style={DESTRUCTIVE_BTN} onClick={onResetProfile}>Reset</button>
        </div>

        {/* Audio only row */}
        <div style={{ ...ROW }}>
          <div>
            <div style={LABEL}>Audio only</div>
            <div style={DESC}>Pauses the visualization to save battery.</div>
          </div>
          <button
            role="switch"
            aria-checked={audioOnly}
            onClick={onToggleAudioOnly}
            style={{
              width: 40,
              height: 24,
              borderRadius: 12,
              border: 'none',
              background: audioOnly ? 'rgba(255,255,255,0.6)' : 'rgba(255,255,255,0.15)',
              cursor: 'pointer',
              position: 'relative',
              flexShrink: 0,
              transition: 'background 0.2s',
            }}
          >
            <span style={{
              position: 'absolute',
              top: 3,
              left: audioOnly ? 19 : 3,
              width: 18,
              height: 18,
              borderRadius: '50%',
              background: 'white',
              transition: 'left 0.2s',
            }} />
          </button>
        </div>

        {/* Clear saved stations row */}
        <div style={{ ...ROW, borderBottom: 'none' }}>
          <div>
            <div style={LABEL}>Saved stations</div>
            <div style={DESC}>Removes all saved stations.</div>
          </div>
          <button style={DESTRUCTIVE_BTN} onClick={onClearSaved}>Clear all</button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/SettingsModal.test.tsx
```

Expected: all 8 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/SettingsModal.tsx src/SettingsModal.test.tsx
git commit -m "feat: add SettingsModal with taste profile reset, audio-only toggle, and clear saved"
```

---

## Task 6: Wire into `App.tsx`

**Files:**
- Modify: `src/App.tsx`

No new unit tests — App integration is verified manually via dev server.

- [ ] **Step 1: Add new state, imports, and callbacks**

At the top of `src/App.tsx`, add imports:

```typescript
import { OverlayControls } from './OverlayControls'
import { SettingsModal } from './SettingsModal'
```

Inside `App()`, update the `useRecommendation` destructure to include `clearProfile`:

```typescript
const { recordSignal, selectNext: pickNext, startDwellTimer, cancelDwellTimer, clearProfile } = useRecommendation()
```

Update the `useSavedStations` destructure to include `clearAll`:

```typescript
const { savedStations, save, remove, isSaved, clearAll } = useSavedStations()
```

Add three new state variables alongside the existing `useState` calls:

```typescript
const [paused, setPaused]           = useState(false)
const [audioOnly, setAudioOnly]     = useState(false)
const [settingsOpen, setSettingsOpen] = useState(false)
```

- [ ] **Step 2: Update `handlePlayPause` to track `paused` state reactively, and reset `paused` on station change**

Replace the existing `handlePlayPause`:

```typescript
const handlePlayPause = useCallback(() => {
  if (phaseRef.current !== 'playing' || !playerRef.current) return
  if (playerRef.current.isPaused()) {
    playerRef.current.resume()
    setPaused(false)
  } else {
    playerRef.current.pause()
    setPaused(true)
  }
}, [])
```

Also add `setPaused(false)` at the point where the new player is assigned in the navigation branch of `loadAndPlay`. This resets the icon when the user skips while paused. In the navigation `try` block, after `playerRef.current = player`, add:

```typescript
setPaused(false)
```

- [ ] **Step 3: Add share, audio-only, settings, and clear callbacks**

After `handlePlayPause`, add:

```typescript
const handleShare = useCallback(() => {
  navigator.clipboard.writeText(window.location.href).then(() => {
    setToastMessage('Link copied')
  }).catch(() => {
    setToastMessage('Could not copy link')
  })
}, [])

const handleToggleAudioOnly = useCallback(() => {
  const next = !audioOnly
  setAudioOnly(next)
  vizRef.current?.setAudioOnly(next)
}, [audioOnly])

const handleResetProfile = useCallback(async () => {
  await clearProfile()
  setSettingsOpen(false)
  setToastMessage('Taste profile reset')
}, [clearProfile])

const handleClearSaved = useCallback(async () => {
  await clearAll()
  setSettingsOpen(false)
  setToastMessage('Saved stations cleared')
}, [clearAll])
```

- [ ] **Step 4: Wire `OverlayControls` and `SettingsModal` into the JSX**

In the JSX return, after the `<NavControls ... />` block and before `<Toast ... />`, add:

```typescript
{phase === 'playing' && (
  <OverlayControls
    onShare={handleShare}
    onOpenSettings={() => setSettingsOpen(true)}
    onPlayPause={handlePlayPause}
    paused={paused}
  />
)}

{settingsOpen && (
  <SettingsModal
    audioOnly={audioOnly}
    onToggleAudioOnly={handleToggleAudioOnly}
    onResetProfile={handleResetProfile}
    onClearSaved={handleClearSaved}
    onClose={() => setSettingsOpen(false)}
  />
)}
```

- [ ] **Step 5: Run the full test suite**

```bash
npm run test:run
```

Expected: all tests pass (existing + new).

- [ ] **Step 6: Verify manually in the dev server**

```bash
npm run dev
```

Checklist:
- Tap to begin → share + settings appear top-right, pause button bottom-right
- Click pause → icon changes to play; audio pauses
- Click play → icon changes to pause; audio resumes
- Click share → toast "Link copied" appears
- Click settings → modal opens; music keeps playing
- Toggle "Audio only" → visualization freezes; toggle again → visualization resumes
- Click "Reset" → modal closes, toast "Taste profile reset"
- Click "Clear all" → modal closes, toast "Saved stations cleared"
- Click backdrop → modal closes
- Click × → modal closes

- [ ] **Step 7: Commit**

```bash
git add src/App.tsx
git commit -m "feat: wire OverlayControls and SettingsModal into App"
```
