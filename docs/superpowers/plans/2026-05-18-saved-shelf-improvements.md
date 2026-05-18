# Saved Shelf Improvements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add sort controls, animated playing indicator, drag-to-reorder, and keyboard shortcuts (F + 1–0) to the saved stations shelf.

**Architecture:** A new `useShelfSort` hook owns sort mode and custom order (both in `localStorage`). It produces the sorted station array that both `SavedShelf` and `useNavigation` consume. `SavedShelf` handles its own drag-and-drop via HTML5 DnD (desktop) and touch events (mobile). `useNavigation` gains an `onPlayByIndex` param and handles `F` + `1`–`0` keys.

**Tech Stack:** React 19 + TypeScript + Vitest + Testing Library — no new dependencies.

---

## File Map

| File | Change |
|---|---|
| `src/useShelfSort.ts` | **Create** — sort mode + order management (localStorage) |
| `src/useShelfSort.test.ts` | **Create** — tests for all sort modes + order helpers |
| `src/SavedShelf.tsx` | **Modify** — sort bar, animated equalizer, drag handles, number badges |
| `src/SavedShelf.test.tsx` | **Create** — tests for new SavedShelf behaviours |
| `src/useNavigation.ts` | **Modify** — add `onPlayByIndex` param, `F` key, `1`–`0` keys |
| `src/useNavigation.test.ts` | **Modify** — add tests for new keyboard shortcuts |
| `src/App.tsx` | **Modify** — wire `useShelfSort`, pass new props to `SavedShelf` + `useNavigation`, call `appendToOrder` on save |

---

## Task 1: `useShelfSort` hook

**Files:**
- Create: `src/useShelfSort.ts`
- Create: `src/useShelfSort.test.ts`

- [ ] **Step 1: Write failing tests**

Create `src/useShelfSort.test.ts`:

```typescript
import { renderHook, act } from '@testing-library/react'
import { useShelfSort } from './useShelfSort'
import type { Station } from './types'

function s(name: string, country: string, tags: string, uuid = name): Station {
  return { stationuuid: uuid, name, url: '', country, tags, bitrate: 128, corsFriendly: false }
}

const stations = [
  s('Zoo Radio', 'Germany', 'jazz'),
  s('Alpha FM', 'France', 'pop'),
  s('Beat Lab', 'Netherlands', 'electronic'),
]

beforeEach(() => localStorage.clear())

describe('useShelfSort', () => {
  it('defaults to saved sort mode', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    expect(result.current.sortMode).toBe('saved')
  })

  it('restores sort mode from localStorage', () => {
    localStorage.setItem('shelf_sort', 'az')
    const { result } = renderHook(() => useShelfSort(stations))
    expect(result.current.sortMode).toBe('az')
  })

  it('az sorts by name ascending', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.setSortMode('az'))
    expect(result.current.sortedStations.map(s => s.name)).toEqual(['Alpha FM', 'Beat Lab', 'Zoo Radio'])
  })

  it('country sorts by normalised country ascending', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.setSortMode('country'))
    expect(result.current.sortedStations.map(s => s.country)).toEqual(['France', 'Germany', 'Netherlands'])
  })

  it('genre sorts by primary tag ascending', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.setSortMode('genre'))
    expect(result.current.sortedStations.map(s => s.tags)).toEqual(['electronic', 'jazz', 'pop'])
  })

  it('saved sort respects custom order from localStorage', () => {
    localStorage.setItem('shelf_order', JSON.stringify(['Beat Lab', 'Zoo Radio', 'Alpha FM']))
    const { result } = renderHook(() => useShelfSort(stations))
    expect(result.current.sortedStations.map(s => s.stationuuid)).toEqual(['Beat Lab', 'Zoo Radio', 'Alpha FM'])
  })

  it('setOrder updates sorted order and persists to localStorage', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.setOrder(['Alpha FM', 'Zoo Radio', 'Beat Lab']))
    expect(result.current.sortedStations.map(s => s.stationuuid)).toEqual(['Alpha FM', 'Zoo Radio', 'Beat Lab'])
    expect(JSON.parse(localStorage.getItem('shelf_order')!)).toEqual(['Alpha FM', 'Zoo Radio', 'Beat Lab'])
  })

  it('appendToOrder appends a new UUID not already in the list', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.appendToOrder('new-uuid'))
    const stored = JSON.parse(localStorage.getItem('shelf_order')!)
    expect(stored[stored.length - 1]).toBe('new-uuid')
  })

  it('appendToOrder is idempotent — does not duplicate existing UUID', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.appendToOrder('Zoo Radio'))
    act(() => result.current.appendToOrder('Zoo Radio'))
    const stored: string[] = JSON.parse(localStorage.getItem('shelf_order')!)
    expect(stored.filter(id => id === 'Zoo Radio')).toHaveLength(1)
  })

  it('stations not in custom order array sort after those that are', () => {
    localStorage.setItem('shelf_order', JSON.stringify(['Alpha FM']))
    const { result } = renderHook(() => useShelfSort(stations))
    expect(result.current.sortedStations[0].stationuuid).toBe('Alpha FM')
  })

  it('setSortMode persists to localStorage', () => {
    const { result } = renderHook(() => useShelfSort(stations))
    act(() => result.current.setSortMode('genre'))
    expect(localStorage.getItem('shelf_sort')).toBe('genre')
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npm run test:run -- --reporter=verbose src/useShelfSort.test.ts
```

Expected: all tests fail with "Cannot find module './useShelfSort'".

- [ ] **Step 3: Implement `useShelfSort`**

Create `src/useShelfSort.ts`:

```typescript
import { useState, useMemo, useCallback } from 'react'
import { normalizeCountry, primaryGenre } from './utils'
import type { Station } from './types'

export type SortMode = 'saved' | 'az' | 'country' | 'genre'

const SORT_KEY = 'shelf_sort'
const ORDER_KEY = 'shelf_order'

function readOrder(): string[] {
  try { return JSON.parse(localStorage.getItem(ORDER_KEY) ?? '[]') } catch { return [] }
}

function writeOrder(order: string[]): void {
  try { localStorage.setItem(ORDER_KEY, JSON.stringify(order)) } catch { /* ignore */ }
}

export function useShelfSort(stations: Station[]) {
  const [sortMode, setSortModeState] = useState<SortMode>(() => {
    const v = localStorage.getItem(SORT_KEY)
    return (v === 'az' || v === 'country' || v === 'genre') ? v : 'saved'
  })

  const [customOrder, setCustomOrder] = useState<string[]>(readOrder)

  const setSortMode = useCallback((mode: SortMode) => {
    setSortModeState(mode)
    try { localStorage.setItem(SORT_KEY, mode) } catch { /* ignore */ }
  }, [])

  const setOrder = useCallback((order: string[]) => {
    setCustomOrder(order)
    writeOrder(order)
  }, [])

  const appendToOrder = useCallback((uuid: string) => {
    setCustomOrder(prev => {
      if (prev.includes(uuid)) return prev
      const next = [...prev, uuid]
      writeOrder(next)
      return next
    })
  }, [])

  const sortedStations = useMemo(() => {
    if (sortMode === 'az') {
      return [...stations].sort((a, b) => a.name.localeCompare(b.name))
    }
    if (sortMode === 'country') {
      return [...stations].sort((a, b) =>
        normalizeCountry(a.country).localeCompare(normalizeCountry(b.country))
      )
    }
    if (sortMode === 'genre') {
      return [...stations].sort((a, b) =>
        primaryGenre(a.tags).localeCompare(primaryGenre(b.tags))
      )
    }
    // 'saved' — custom drag order; stations absent from the order list fall to end
    const orderMap = new Map(customOrder.map((uuid, i) => [uuid, i]))
    return [...stations].sort((a, b) => {
      const ai = orderMap.get(a.stationuuid) ?? Infinity
      const bi = orderMap.get(b.stationuuid) ?? Infinity
      return ai - bi
    })
  }, [stations, sortMode, customOrder])

  return { sortMode, setSortMode, sortedStations, setOrder, appendToOrder }
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- --reporter=verbose src/useShelfSort.test.ts
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useShelfSort.ts src/useShelfSort.test.ts
git commit -m "feat: useShelfSort hook — sort modes + custom order (localStorage)"
```

---

## Task 2: `SavedShelf` — sort bar + animated playing indicator

**Files:**
- Create: `src/SavedShelf.test.tsx`
- Modify: `src/SavedShelf.tsx`

- [ ] **Step 1: Write failing tests**

Create `src/SavedShelf.test.tsx`:

```tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { SavedShelf } from './SavedShelf'
import type { Station } from './types'
import type { SortMode } from './useShelfSort'

function s(name: string, uuid = name): Station {
  return { stationuuid: uuid, name, url: '', country: 'Germany', tags: 'jazz', bitrate: 128, corsFriendly: false }
}

const stations = [s('Alpha FM'), s('Beat Lab'), s('Zoo Radio')]
const noop = () => {}
const noopAsync = async () => {}

function shelf(props: Partial<Parameters<typeof SavedShelf>[0]> = {}) {
  return render(
    <SavedShelf
      stations={stations}
      currentStation={null}
      sortMode="saved"
      onSortChange={noop}
      onReorder={noop}
      onPlay={noop}
      onRemove={noopAsync}
      onClose={noop}
      {...props}
    />
  )
}

describe('SavedShelf', () => {
  it('renders sort chips for all four modes', () => {
    shelf()
    expect(screen.getByText('Saved')).toBeInTheDocument()
    expect(screen.getByText('A–Z')).toBeInTheDocument()
    expect(screen.getByText('Country')).toBeInTheDocument()
    expect(screen.getByText('Genre')).toBeInTheDocument()
  })

  it('calls onSortChange with the correct mode when a chip is clicked', () => {
    const onSortChange = vi.fn()
    shelf({ onSortChange })
    fireEvent.click(screen.getByText('A–Z'))
    expect(onSortChange).toHaveBeenCalledWith('az')
  })

  it('shows equalizer indicator on the currently playing station row', () => {
    shelf({ currentStation: stations[1] })
    expect(screen.getByTestId('now-playing-indicator')).toBeInTheDocument()
  })

  it('does not show equalizer when no station is current', () => {
    shelf({ currentStation: null })
    expect(screen.queryByTestId('now-playing-indicator')).not.toBeInTheDocument()
  })

  it('renders all station names', () => {
    shelf()
    for (const st of stations) expect(screen.getByText(st.name)).toBeInTheDocument()
  })

  it('calls onPlay when a row is clicked', () => {
    const onPlay = vi.fn()
    shelf({ onPlay })
    fireEvent.click(screen.getByText('Alpha FM'))
    expect(onPlay).toHaveBeenCalledWith(stations[0])
  })

  it('calls onRemove when the × button is clicked', () => {
    const onRemove = vi.fn()
    shelf({ onRemove })
    fireEvent.click(screen.getByRole('button', { name: 'Remove Alpha FM' }))
    expect(onRemove).toHaveBeenCalledWith('Alpha FM')
  })

  it('calls onClose when the scrim is clicked', () => {
    const onClose = vi.fn()
    shelf({ onClose })
    fireEvent.click(screen.getByTestId('shelf-scrim'))
    expect(onClose).toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- --reporter=verbose src/SavedShelf.test.tsx
```

Expected: failures because `sortMode`, `onSortChange`, `onReorder` props don't exist yet and `now-playing-indicator` testid is absent.

- [ ] **Step 3: Update `SavedShelf.tsx` with sort bar + playing indicator**

Replace the full file:

```tsx
import { useEffect, useRef, useState } from 'react'
import { normalizeCountry, primaryGenre } from './utils'
import type { Station } from './types'
import type { SortMode } from './useShelfSort'

interface SavedShelfProps {
  stations: Station[]
  currentStation: Station | null
  sortMode: SortMode
  onSortChange: (mode: SortMode) => void
  onReorder: (uuids: string[]) => void
  onPlay: (station: Station) => void
  onRemove: (stationuuid: string) => void
  onClose: () => void
}

const SORT_CHIPS: { label: string; mode: SortMode }[] = [
  { label: 'Saved',   mode: 'saved'   },
  { label: 'A–Z',     mode: 'az'      },
  { label: 'Country', mode: 'country' },
  { label: 'Genre',   mode: 'genre'   },
]

function NowPlayingIndicator() {
  return (
    <div
      data-testid="now-playing-indicator"
      style={{ display: 'flex', alignItems: 'flex-end', gap: 2, marginRight: 8, height: 12, flexShrink: 0 }}
    >
      {[0, 1, 2].map(i => (
        <div key={i} style={{
          width: 3,
          background: 'rgba(255,255,255,0.55)',
          borderRadius: 1,
          animation: `eq-bar 0.9s ease-in-out ${i * 0.2}s infinite`,
        }} />
      ))}
    </div>
  )
}

export function SavedShelf({
  stations, currentStation, sortMode, onSortChange, onReorder, onPlay, onRemove, onClose,
}: SavedShelfProps) {
  const [entered, setEntered] = useState(false)
  const [dragIndex, setDragIndex] = useState<number | null>(null)
  const [dragOverIndex, setDragOverIndex] = useState<number | null>(null)
  const touchDragRef = useRef<{ startIndex: number; currentIndex: number }>({ startIndex: -1, currentIndex: -1 })
  const isDesktop = navigator.maxTouchPoints === 0

  useEffect(() => {
    const id = requestAnimationFrame(() => setEntered(true))
    return () => cancelAnimationFrame(id)
  }, [])

  function commitReorder(fromIndex: number, toIndex: number) {
    if (fromIndex === toIndex || fromIndex < 0 || toIndex < 0) return
    const reordered = [...stations]
    const [item] = reordered.splice(fromIndex, 1)
    reordered.splice(toIndex, 0, item)
    onReorder(reordered.map(s => s.stationuuid))
  }

  return (
    <>
      <style>{`@keyframes eq-bar { 0%,100%{height:3px} 50%{height:10px} }`}</style>
      <div
        data-testid="shelf-scrim"
        onClick={onClose}
        style={{
          position: 'fixed', inset: 0,
          background: 'rgba(0,0,0,0.45)',
          zIndex: 10,
        }}
      />
      <div
        role="dialog"
        aria-label="Saved stations"
        style={{
          position: 'fixed',
          bottom: 0, left: 0, right: 0,
          maxHeight: '85vh',
          background: 'rgba(12,12,12,0.96)',
          backdropFilter: 'blur(20px)',
          WebkitBackdropFilter: 'blur(20px)',
          borderRadius: '16px 16px 0 0',
          zIndex: 11,
          display: 'flex',
          flexDirection: 'column',
          overflow: 'hidden',
          transform: entered ? 'translateY(0)' : 'translateY(100%)',
          transition: 'transform 300ms cubic-bezier(0.4, 0, 0.2, 1)',
        }}
      >
        <div style={{
          width: 36, height: 4,
          background: 'rgba(255,255,255,0.18)',
          borderRadius: 2,
          margin: '12px auto 8px',
          flexShrink: 0,
        }} />

        {/* Sort chips */}
        <div style={{
          display: 'flex',
          gap: 8,
          padding: '0 16px 12px',
          flexShrink: 0,
        }}>
          {SORT_CHIPS.map(({ label, mode }) => (
            <button
              key={mode}
              onClick={() => onSortChange(mode)}
              style={{
                background: 'none',
                border: sortMode === mode ? '1px solid rgba(255,255,255,0.2)' : '1px solid transparent',
                borderRadius: 20,
                padding: '4px 12px',
                cursor: 'pointer',
                fontFamily: 'system-ui, sans-serif',
                fontSize: 12,
                color: sortMode === mode ? 'rgba(255,255,255,0.85)' : 'rgba(255,255,255,0.30)',
                transition: 'color 150ms, border-color 150ms',
              }}
            >
              {label}
            </button>
          ))}
        </div>

        <div style={{ overflowY: 'auto', flex: 1, padding: '4px 0 40px' }}>
          {stations.length === 0 ? (
            <div style={{
              textAlign: 'center',
              marginTop: 64,
              fontFamily: 'system-ui, sans-serif',
              fontSize: 14,
              color: 'rgba(255,255,255,0.28)',
              letterSpacing: '0.02em',
            }}>
              No saved stations yet
            </div>
          ) : (
            stations.map((s, i) => {
              const isCurrent = currentStation?.stationuuid === s.stationuuid
              const isDragging = dragIndex === i
              const isOver = dragOverIndex === i && dragIndex !== i
              const showDragHandle = sortMode === 'saved'
              const showNumber = isDesktop && i < 10

              return (
                <div
                  key={s.stationuuid}
                  data-index={i}
                  draggable={showDragHandle}
                  onDragStart={() => setDragIndex(i)}
                  onDragOver={e => { e.preventDefault(); setDragOverIndex(i) }}
                  onDrop={() => { commitReorder(dragIndex!, i); setDragIndex(null); setDragOverIndex(null) }}
                  onDragEnd={() => { setDragIndex(null); setDragOverIndex(null) }}
                  onTouchStart={() => { touchDragRef.current = { startIndex: i, currentIndex: i } }}
                  onTouchMove={e => {
                    e.preventDefault()
                    const touch = e.touches[0]
                    const el = document.elementFromPoint(touch.clientX, touch.clientY)
                    const row = el?.closest('[data-index]') as HTMLElement | null
                    if (row) {
                      const idx = parseInt(row.dataset.index ?? '-1', 10)
                      if (idx >= 0) touchDragRef.current.currentIndex = idx
                    }
                  }}
                  onTouchEnd={() => {
                    const { startIndex, currentIndex } = touchDragRef.current
                    commitReorder(startIndex, currentIndex)
                  }}
                  onClick={() => onPlay(s)}
                  style={{
                    display: 'flex',
                    alignItems: 'center',
                    padding: '11px 20px',
                    cursor: 'pointer',
                    background: isCurrent
                      ? 'rgba(255,255,255,0.05)'
                      : isOver
                        ? 'rgba(255,255,255,0.04)'
                        : 'transparent',
                    opacity: isDragging ? 0.4 : 1,
                    transition: 'opacity 150ms, background 150ms',
                  }}
                >
                  {showDragHandle && (
                    <span style={{
                      marginRight: 12,
                      color: 'rgba(255,255,255,0.20)',
                      fontSize: 14,
                      cursor: 'grab',
                      flexShrink: 0,
                      userSelect: 'none',
                    }}>
                      ⠿
                    </span>
                  )}

                  {isCurrent && <NowPlayingIndicator />}

                  <div style={{ flex: 1, minWidth: 0 }}>
                    <div style={{
                      fontFamily: 'system-ui, sans-serif',
                      fontSize: 15,
                      fontWeight: 500,
                      color: 'rgba(255,255,255,0.85)',
                      whiteSpace: 'nowrap',
                      overflow: 'hidden',
                      textOverflow: 'ellipsis',
                    }}>
                      {s.name}
                    </div>
                    <div style={{
                      marginTop: 2,
                      fontFamily: 'system-ui, sans-serif',
                      fontSize: 12,
                      color: 'rgba(255,255,255,0.32)',
                      letterSpacing: '0.02em',
                      whiteSpace: 'nowrap',
                      overflow: 'hidden',
                      textOverflow: 'ellipsis',
                    }}>
                      {normalizeCountry(s.country)}{s.tags ? ` · ${primaryGenre(s.tags)}` : ''}
                    </div>
                  </div>

                  {showNumber && (
                    <span style={{
                      fontFamily: 'system-ui, sans-serif',
                      fontSize: 10,
                      color: 'rgba(255,255,255,0.20)',
                      marginLeft: 8,
                      marginRight: 4,
                      flexShrink: 0,
                      userSelect: 'none',
                    }}>
                      {i === 9 ? '0' : String(i + 1)}
                    </span>
                  )}

                  <button
                    aria-label={`Remove ${s.name}`}
                    onClick={e => { e.stopPropagation(); onRemove(s.stationuuid) }}
                    style={{
                      background: 'none',
                      border: 'none',
                      cursor: 'pointer',
                      color: 'rgba(255,255,255,0.28)',
                      fontSize: 20,
                      lineHeight: 1,
                      padding: '4px 8px',
                      marginLeft: 8,
                      flexShrink: 0,
                    }}
                  >
                    ×
                  </button>
                </div>
              )
            })
          )}
        </div>
      </div>
    </>
  )
}
```

- [ ] **Step 4: Run tests**

```bash
npm run test:run -- --reporter=verbose src/SavedShelf.test.tsx
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/SavedShelf.tsx src/SavedShelf.test.tsx
git commit -m "feat: SavedShelf — sort bar, animated now-playing indicator, drag-to-reorder, number badges"
```

---

## Task 3: Update `useNavigation` — F key + 1–0 shortcuts

**Files:**
- Modify: `src/useNavigation.ts`
- Modify: `src/useNavigation.test.ts`

- [ ] **Step 1: Write failing tests**

Append to `src/useNavigation.test.ts` (after the last existing test):

```typescript
describe('F key shortcut', () => {
  it('opens shelf when closed and phase is playing', () => {
    const onOpenShelf = vi.fn()
    renderHook(() => useNavigation(
      'playing', vi.fn(), vi.fn(), true, vi.fn(), onOpenShelf, vi.fn(), false, vi.fn(),
      vi.fn(), [],
    ))
    act(() => fireKey('f'))
    expect(onOpenShelf).toHaveBeenCalledTimes(1)
  })

  it('closes shelf when open and F is pressed', () => {
    const onCloseShelf = vi.fn()
    renderHook(() => useNavigation(
      'playing', vi.fn(), vi.fn(), true, vi.fn(), vi.fn(), onCloseShelf, true, vi.fn(),
      vi.fn(), [],
    ))
    act(() => fireKey('f'))
    expect(onCloseShelf).toHaveBeenCalledTimes(1)
  })

  it('ignores F key when phase is not playing', () => {
    const onOpenShelf = vi.fn()
    renderHook(() => useNavigation(
      'idle', vi.fn(), vi.fn(), true, vi.fn(), onOpenShelf, vi.fn(), false, vi.fn(),
      vi.fn(), [],
    ))
    act(() => fireKey('f'))
    expect(onOpenShelf).not.toHaveBeenCalled()
  })
})

describe('number key shortcuts (shelf open)', () => {
  it('calls onPlayByIndex(0) when 1 is pressed and shelf is open', () => {
    const onPlayByIndex = vi.fn()
    const stations = [{ stationuuid: 'a' }] as any
    renderHook(() => useNavigation(
      'playing', vi.fn(), vi.fn(), true, vi.fn(), vi.fn(), vi.fn(), true, vi.fn(),
      onPlayByIndex, stations,
    ))
    act(() => fireKey('1'))
    expect(onPlayByIndex).toHaveBeenCalledWith(0)
  })

  it('calls onPlayByIndex(9) when 0 is pressed', () => {
    const onPlayByIndex = vi.fn()
    const stations = Array.from({ length: 10 }, (_, i) => ({ stationuuid: String(i) })) as any
    renderHook(() => useNavigation(
      'playing', vi.fn(), vi.fn(), true, vi.fn(), vi.fn(), vi.fn(), true, vi.fn(),
      onPlayByIndex, stations,
    ))
    act(() => fireKey('0'))
    expect(onPlayByIndex).toHaveBeenCalledWith(9)
  })

  it('does not call onPlayByIndex when shelf is closed', () => {
    const onPlayByIndex = vi.fn()
    renderHook(() => useNavigation(
      'playing', vi.fn(), vi.fn(), true, vi.fn(), vi.fn(), vi.fn(), false, vi.fn(),
      onPlayByIndex, [],
    ))
    act(() => fireKey('1'))
    expect(onPlayByIndex).not.toHaveBeenCalled()
  })

  it('does not call onPlayByIndex when index is out of range', () => {
    const onPlayByIndex = vi.fn()
    renderHook(() => useNavigation(
      'playing', vi.fn(), vi.fn(), true, vi.fn(), vi.fn(), vi.fn(), true, vi.fn(),
      onPlayByIndex, [{ stationuuid: 'a' }] as any,
    ))
    act(() => fireKey('2'))
    expect(onPlayByIndex).not.toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npm run test:run -- --reporter=verbose src/useNavigation.test.ts
```

Expected: the new tests fail because `useNavigation` doesn't accept the two new params yet.

- [ ] **Step 3: Update `useNavigation.ts`**

Replace the full file:

```typescript
import { useEffect, useRef, useState } from 'react'
import type { Phase, Station } from './types'

export type FlashEvent = { direction: 'left' | 'right' | 'up' | 'down' }

export function useNavigation(
  phase: Phase,
  onNext: () => void,
  onPrevious: () => void,
  canGoPrevious: boolean,
  onSave: () => void,
  onOpenShelf: () => void,
  onCloseShelf: () => void,
  isShelfOpen: boolean,
  onPlayPause: () => void,
  onPlayByIndex: (index: number) => void = () => {},
  sortedSavedStations: Station[] = [],
): { keyboardFlash: FlashEvent | null } {
  const phaseRef = useRef(phase)
  const onNextRef = useRef(onNext)
  const onPreviousRef = useRef(onPrevious)
  const canGoPreviousRef = useRef(canGoPrevious)
  const onSaveRef = useRef(onSave)
  const onOpenShelfRef = useRef(onOpenShelf)
  const onCloseShelfRef = useRef(onCloseShelf)
  const isShelfOpenRef = useRef(isShelfOpen)
  const onPlayPauseRef = useRef(onPlayPause)
  const onPlayByIndexRef = useRef(onPlayByIndex)
  const sortedSavedStationsRef = useRef(sortedSavedStations)

  useEffect(() => { phaseRef.current = phase }, [phase])
  useEffect(() => { onNextRef.current = onNext }, [onNext])
  useEffect(() => { onPreviousRef.current = onPrevious }, [onPrevious])
  useEffect(() => { canGoPreviousRef.current = canGoPrevious }, [canGoPrevious])
  useEffect(() => { onSaveRef.current = onSave }, [onSave])
  useEffect(() => { onOpenShelfRef.current = onOpenShelf }, [onOpenShelf])
  useEffect(() => { onCloseShelfRef.current = onCloseShelf }, [onCloseShelf])
  useEffect(() => { isShelfOpenRef.current = isShelfOpen }, [isShelfOpen])
  useEffect(() => { onPlayPauseRef.current = onPlayPause }, [onPlayPause])
  useEffect(() => { onPlayByIndexRef.current = onPlayByIndex }, [onPlayByIndex])
  useEffect(() => { sortedSavedStationsRef.current = sortedSavedStations }, [sortedSavedStations])

  const [keyboardFlash, setKeyboardFlash] = useState<FlashEvent | null>(null)
  const flashTimeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    const isDesktop = navigator.maxTouchPoints === 0

    const handleKeyDown = (e: KeyboardEvent) => {
      if (phaseRef.current !== 'playing') return

      let direction: FlashEvent['direction'] | null = null

      if (e.key === 'ArrowRight') {
        direction = 'right'
        onNextRef.current()
      } else if (e.key === 'ArrowLeft' && canGoPreviousRef.current) {
        direction = 'left'
        onPreviousRef.current()
      } else if (e.key === 'ArrowUp') {
        direction = 'up'
        onOpenShelfRef.current()
      } else if (e.key === 'ArrowDown') {
        direction = 'down'
        if (isShelfOpenRef.current) {
          onCloseShelfRef.current()
        } else {
          onSaveRef.current()
        }
      } else if (e.key === ' ') {
        e.preventDefault()
        onPlayPauseRef.current()
      } else if (e.key === 'Escape' && isShelfOpenRef.current) {
        onCloseShelfRef.current()
      } else if (isDesktop && (e.key === 'f' || e.key === 'F')) {
        if (isShelfOpenRef.current) {
          onCloseShelfRef.current()
        } else {
          onOpenShelfRef.current()
        }
      } else if (isDesktop && isShelfOpenRef.current) {
        const num = e.key >= '1' && e.key <= '9'
          ? parseInt(e.key, 10) - 1
          : e.key === '0' ? 9 : -1
        if (num >= 0 && num < sortedSavedStationsRef.current.length) {
          e.preventDefault()
          onPlayByIndexRef.current(num)
        }
      }

      if (direction) {
        e.preventDefault()
        if (flashTimeoutRef.current) clearTimeout(flashTimeoutRef.current)
        setKeyboardFlash({ direction })
        flashTimeoutRef.current = setTimeout(() => setKeyboardFlash(null), 180)
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => {
      window.removeEventListener('keydown', handleKeyDown)
      if (flashTimeoutRef.current) clearTimeout(flashTimeoutRef.current)
    }
  }, [])

  return { keyboardFlash }
}
```

- [ ] **Step 4: Run tests**

```bash
npm run test:run -- --reporter=verbose src/useNavigation.test.ts
```

Expected: all tests pass (new + existing).

- [ ] **Step 5: Commit**

```bash
git add src/useNavigation.ts src/useNavigation.test.ts
git commit -m "feat: useNavigation — F key toggles shelf, 1–0 play by index"
```

---

## Task 4: Wire everything in `App.tsx`

**Files:**
- Modify: `src/App.tsx`

- [ ] **Step 1: Add `useShelfSort` import**

In `src/App.tsx`, add to the imports block (after the existing imports):

```typescript
import { useShelfSort } from './useShelfSort'
```

- [ ] **Step 2: Instantiate `useShelfSort` after `useSavedStations`**

Find (line ~84):
```typescript
const { savedStations, save, remove, isSaved, clearAll } = useSavedStations()
```

Add immediately after:
```typescript
const { sortMode, setSortMode, sortedStations: sortedSavedStations, setOrder, appendToOrder } = useShelfSort(savedStations)
```

- [ ] **Step 3: Call `appendToOrder` on successful save**

Find in `handleSave` (line ~212):
```typescript
    const result = await save(station)
    if (result === 'saved') {
      vizRef.current?.triggerBurst()
```

Replace with:
```typescript
    const result = await save(station)
    if (result === 'saved') {
      appendToOrder(station.stationuuid)
      vizRef.current?.triggerBurst()
```

- [ ] **Step 4: Add `onPlayByIndex` callback**

Find the `openShelf` / `closeShelf` definitions (line ~279), and add after `closeShelf`:

```typescript
const onPlayByIndex = useCallback((i: number) => {
  const s = sortedSavedStations[i]
  if (s) { playStation(s); closeShelf() }
}, [sortedSavedStations, playStation, closeShelf])
```

- [ ] **Step 5: Update the `useNavigation` call**

Find (line ~282):
```typescript
  const { keyboardFlash } = useNavigation(
    phase, handleNext, handlePrevious, canGoPrevious,
    handleSave, openShelf, closeShelf, shelfOpen, handlePlayPause,
  )
```

Replace with:
```typescript
  const { keyboardFlash } = useNavigation(
    phase, handleNext, handlePrevious, canGoPrevious,
    handleSave, openShelf, closeShelf, shelfOpen, handlePlayPause,
    onPlayByIndex, sortedSavedStations,
  )
```

- [ ] **Step 6: Update the `SavedShelf` render**

Find (line ~527):
```tsx
      {shelfOpen && (
        <SavedShelf
          stations={savedStations}
          currentStation={station}
          onPlay={s => { playStation(s); closeShelf() }}
          onRemove={remove}
          onClose={closeShelf}
        />
      )}
```

Replace with:
```tsx
      {shelfOpen && (
        <SavedShelf
          stations={sortedSavedStations}
          currentStation={station}
          sortMode={sortMode}
          onSortChange={setSortMode}
          onReorder={setOrder}
          onPlay={s => { playStation(s); closeShelf() }}
          onRemove={remove}
          onClose={closeShelf}
        />
      )}
```

- [ ] **Step 7: Run the full test suite**

```bash
npm run test:run
```

Expected: all tests pass (328+).

- [ ] **Step 8: Commit**

```bash
git add src/App.tsx
git commit -m "feat: wire useShelfSort and keyboard shortcuts into App"
```

---

## Task 5: Update design spec Section 17

**Files:**
- Modify: `/Users/vitorfreitas/dev/spectrale/docs/web-design-spec.md`

- [ ] **Step 1: Mark feature complete in Section 17**

Find the row:
```
| Saved shelf improvements | ❌ Not started — current station highlighted with a speaker icon; sort options: Saved order (default) · A–Z · Country · Genre; sort preference persisted to IDB |
```

Replace with:
```
| Saved shelf improvements | ✅ Done — animated equalizer on playing station row; sort chips (Saved · A–Z · Country · Genre, persisted to localStorage); drag-to-reorder in Saved mode (HTML5 DnD desktop + touch); F key opens/closes shelf; 1–0 keys play first 10 stations with visible number badges (desktop only) |
```

- [ ] **Step 2: Commit the spec update**

```bash
cd /Users/vitorfreitas/dev/spectrale && git add docs/web-design-spec.md && git commit -m "docs: mark saved shelf improvements as done"
cd /Users/vitorfreitas/dev/spectrale-web
```
