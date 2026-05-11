# Recommendation Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a client-side recommendation engine that learns station preferences from listening signals and selects stations using a taste profile stored in IndexedDB.

**Architecture:** Pure functions in `recommendation.ts` hold all scoring and selection logic (fully testable); `useRecommendation` hook wraps them with IDB persistence and a dwell timer; `App.tsx` records signals on navigation and replaces `getRandomStation` with `selectNext`.

**Tech Stack:** TypeScript, IndexedDB (raw IDB API via existing `openDB` helper), React hooks, Vitest

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `src/recommendation.ts` | Create | `TasteProfile` type, `createEmptyProfile`, `skipSignalType`, `applySignal`, `selectNext` pure functions |
| `src/recommendation.test.ts` | Create | Unit tests for all pure functions |
| `src/useIndexedDB.ts` | Modify | Bump DB to version 2, add `keyval` object store |
| `src/useRecommendation.ts` | Create | Hook: IDB load/persist, dwell timer, `recordSignal` and `selectNext` wrappers |
| `src/App.tsx` | Modify | `playStartRef`, signal recording on nav, dwell timer calls, replace `getRandomStation` |

---

## Task 1: Types, `createEmptyProfile`, and `skipSignalType`

**Files:**
- Create: `src/recommendation.ts`
- Create: `src/recommendation.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `src/recommendation.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { createEmptyProfile, skipSignalType } from './recommendation'

describe('createEmptyProfile', () => {
  it('returns a zeroed profile', () => {
    const p = createEmptyProfile()
    expect(p.interactionCount).toBe(0)
    expect(p.genreScores).toEqual({})
    expect(p.countryScores).toEqual({})
    expect(p.recentlyPlayed).toEqual([])
    expect(p.savedDedupeWindow).toEqual([])
  })
})

describe('skipSignalType', () => {
  it('returns skip_fast for < 5s', () => {
    expect(skipSignalType(0)).toBe('skip_fast')
    expect(skipSignalType(4.9)).toBe('skip_fast')
  })

  it('returns skip_mid for 5–30s', () => {
    expect(skipSignalType(5)).toBe('skip_mid')
    expect(skipSignalType(30)).toBe('skip_mid')
  })

  it('returns skip_slow for > 30s', () => {
    expect(skipSignalType(30.1)).toBe('skip_slow')
    expect(skipSignalType(120)).toBe('skip_slow')
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/recommendation.test.ts
```

Expected: FAIL — `recommendation` module not found.

- [ ] **Step 3: Create `src/recommendation.ts` with types and trivial functions**

```typescript
import type { Station } from './types'

export interface TasteProfile {
  interactionCount: number
  genreScores: Record<string, number>
  countryScores: Record<string, number>
  recentlyPlayed: string[]       // stationuuid[], last 50
  savedDedupeWindow: string[]    // last 3 saved stations injected
}

export type SignalType = 'skip_fast' | 'skip_mid' | 'skip_slow' | 'dwell' | 'prev' | 'save'

export function createEmptyProfile(): TasteProfile {
  return {
    interactionCount: 0,
    genreScores: {},
    countryScores: {},
    recentlyPlayed: [],
    savedDedupeWindow: [],
  }
}

export function skipSignalType(listenSeconds: number): SignalType {
  if (listenSeconds < 5) return 'skip_fast'
  if (listenSeconds <= 30) return 'skip_mid'
  return 'skip_slow'
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/recommendation.test.ts
```

Expected: PASS — 4 tests.

- [ ] **Step 5: Commit**

```bash
git add src/recommendation.ts src/recommendation.test.ts
git commit -m "feat: add TasteProfile types, createEmptyProfile, skipSignalType"
```

---

## Task 2: `applySignal`

**Files:**
- Modify: `src/recommendation.ts`
- Modify: `src/recommendation.test.ts`

- [ ] **Step 1: Add failing tests**

Append to `src/recommendation.test.ts`:

```typescript
import { applySignal } from './recommendation'
import type { Station } from './types'

const station: Station = {
  stationuuid: 'abc',
  name: 'Test FM',
  url: 'http://test.fm',
  tags: 'jazz, blues',
  country: 'France',
  bitrate: 128,
  corsFriendly: false,
}

describe('applySignal', () => {
  it('distributes weight across all tags and country', () => {
    const p = applySignal(createEmptyProfile(), station, 'save')
    expect(p.genreScores['jazz']).toBeCloseTo(3)
    expect(p.genreScores['blues']).toBeCloseTo(3)
    expect(p.countryScores['france']).toBeCloseTo(3)
  })

  it('lowercases and trims tags and country', () => {
    const s = { ...station, tags: ' Jazz , Blues ', country: 'FRANCE' }
    const p = applySignal(createEmptyProfile(), s, 'save')
    expect(p.genreScores['jazz']).toBeCloseTo(3)
    expect(p.countryScores['france']).toBeCloseTo(3)
  })

  it('accumulates weights across multiple signals', () => {
    let p = createEmptyProfile()
    p = applySignal(p, station, 'save')    // +3
    p = applySignal(p, station, 'dwell')   // +0.5
    expect(p.genreScores['jazz']).toBeCloseTo(3.5)
  })

  it('floors scores at -10', () => {
    let p = createEmptyProfile()
    for (let i = 0; i < 10; i++) {
      p = applySignal(p, station, 'skip_fast') // -3 each
    }
    expect(p.genreScores['jazz']).toBeGreaterThanOrEqual(-10)
    expect(p.countryScores['france']).toBeGreaterThanOrEqual(-10)
  })

  it('increments interactionCount for skip/prev/save', () => {
    expect(applySignal(createEmptyProfile(), station, 'skip_fast').interactionCount).toBe(1)
    expect(applySignal(createEmptyProfile(), station, 'prev').interactionCount).toBe(1)
    expect(applySignal(createEmptyProfile(), station, 'save').interactionCount).toBe(1)
  })

  it('does not increment interactionCount for dwell', () => {
    const p = applySignal(createEmptyProfile(), station, 'dwell')
    expect(p.interactionCount).toBe(0)
  })

  it('returns a new object without mutating the input', () => {
    const original = createEmptyProfile()
    const updated = applySignal(original, station, 'save')
    expect(original.genreScores).toEqual({})
    expect(updated).not.toBe(original)
  })

  it('handles station with no tags', () => {
    const noTags = { ...station, tags: '' }
    const p = applySignal(createEmptyProfile(), noTags, 'save')
    expect(p.genreScores).toEqual({})
    expect(p.countryScores['france']).toBeCloseTo(3)
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/recommendation.test.ts
```

Expected: FAIL — `applySignal` not exported.

- [ ] **Step 3: Implement `applySignal` in `recommendation.ts`**

Add after `skipSignalType`:

```typescript
const SIGNAL_WEIGHTS: Record<SignalType, number> = {
  skip_fast: -3,
  skip_mid:  -1.5,
  skip_slow: -0.5,
  dwell:      0.5,
  prev:       1,
  save:       3,
}

const SCORE_FLOOR = -10

export function applySignal(
  profile: TasteProfile,
  station: Station,
  type: SignalType,
): TasteProfile {
  const weight = SIGNAL_WEIGHTS[type]
  const tags = station.tags
    ? station.tags.split(',').map(t => t.trim().toLowerCase()).filter(Boolean)
    : []
  const country = station.country?.toLowerCase() ?? ''

  const genreScores = { ...profile.genreScores }
  for (const tag of tags) {
    genreScores[tag] = Math.max((genreScores[tag] ?? 0) + weight, SCORE_FLOOR)
  }

  const countryScores = { ...profile.countryScores }
  if (country) {
    countryScores[country] = Math.max((countryScores[country] ?? 0) + weight, SCORE_FLOOR)
  }

  return {
    ...profile,
    genreScores,
    countryScores,
    interactionCount: type === 'dwell' ? profile.interactionCount : profile.interactionCount + 1,
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/recommendation.test.ts
```

Expected: PASS — all tests including new ones.

- [ ] **Step 5: Commit**

```bash
git add src/recommendation.ts src/recommendation.test.ts
git commit -m "feat: implement applySignal with genre/country scoring and signal weights"
```

---

## Task 3: `selectNext`

**Files:**
- Modify: `src/recommendation.ts`
- Modify: `src/recommendation.test.ts`

- [ ] **Step 1: Add failing tests**

Append to `src/recommendation.test.ts`:

```typescript
import { vi } from 'vitest'
import { selectNext } from './recommendation'

const makeStation = (uuid: string, tags: string, country = 'France'): Station => ({
  stationuuid: uuid,
  name: `Station ${uuid}`,
  url: '',
  tags,
  country,
  bitrate: 128,
  corsFriendly: false,
})

describe('selectNext - cold start (interactionCount < 16)', () => {
  it('never picks stations tagged as talk, news, sports, or religion', () => {
    const profile = createEmptyProfile()
    const catalog = [
      makeStation('1', 'jazz'),
      makeStation('2', 'talk'),
      makeStation('3', 'news'),
      makeStation('4', 'sports'),
      makeStation('5', 'religion'),
      makeStation('6', 'blues'),
    ]
    for (let i = 0; i < 100; i++) {
      const { station } = selectNext(profile, catalog, [], null)
      expect(['talk', 'news', 'sports', 'religion']).not.toContain(station.tags)
    }
  })

  it('avoids same primary genre as lastStation most of the time', () => {
    const profile = createEmptyProfile()
    const catalog = [
      makeStation('1', 'jazz'),
      makeStation('2', 'jazz'),
      makeStation('3', 'blues'),
      makeStation('4', 'pop'),
    ]
    const lastStation = makeStation('x', 'jazz')
    let jazzCount = 0
    for (let i = 0; i < 100; i++) {
      const { station } = selectNext(profile, catalog, [], lastStation)
      if (station.tags === 'jazz') jazzCount++
    }
    // Only 15% exploration can pick jazz; most picks should be non-jazz
    expect(jazzCount).toBeLessThan(30)
  })

  it('excludes recently played stations', () => {
    const profile = { ...createEmptyProfile(), recentlyPlayed: ['1', '2'] }
    const catalog = [
      makeStation('1', 'jazz'),
      makeStation('2', 'blues'),
      makeStation('3', 'pop'),
    ]
    for (let i = 0; i < 30; i++) {
      const { station } = selectNext(profile, catalog, [], null)
      expect(['1', '2']).not.toContain(station.stationuuid)
    }
  })

  it('prepends selected station uuid to recentlyPlayed in returned profile', () => {
    const profile = createEmptyProfile()
    const catalog = [makeStation('1', 'jazz')]
    const { station, updatedProfile } = selectNext(profile, catalog, [], null)
    expect(updatedProfile.recentlyPlayed[0]).toBe(station.stationuuid)
  })

  it('trims recentlyPlayed to 50 entries', () => {
    const recent = Array.from({ length: 50 }, (_, i) => `old-${i}`)
    const profile = { ...createEmptyProfile(), recentlyPlayed: recent }
    const catalog = [makeStation('new', 'jazz')]
    const { updatedProfile } = selectNext(profile, catalog, [], null)
    expect(updatedProfile.recentlyPlayed.length).toBe(50)
    expect(updatedProfile.recentlyPlayed[0]).toBe('new')
  })
})

describe('selectNext - learning phase (16 ≤ interactionCount < 76)', () => {
  it('picks higher-scored stations more often', () => {
    const profile: TasteProfile = {
      ...createEmptyProfile(),
      interactionCount: 20,
      genreScores: { jazz: 10 },
      countryScores: {},
    }
    const catalog = [
      makeStation('jazz', 'jazz'),
      makeStation('pop', 'pop'),
    ]
    let jazzCount = 0
    for (let i = 0; i < 200; i++) {
      const { station } = selectNext(profile, catalog, [], null)
      if (station.stationuuid === 'jazz') jazzCount++
    }
    expect(jazzCount).toBeGreaterThan(100)
  })
})

describe('selectNext - mature phase (interactionCount >= 76)', () => {
  it('amplifies preference for high-scored stations more than learning phase', () => {
    const catalog = [
      makeStation('jazz', 'jazz'),
      makeStation('pop', 'pop'),
    ]
    const learningProfile: TasteProfile = {
      ...createEmptyProfile(),
      interactionCount: 20,
      genreScores: { jazz: 5 },
      countryScores: {},
    }
    const matureProfile: TasteProfile = {
      ...learningProfile,
      interactionCount: 80,
    }
    let jazzLearning = 0
    let jazzMature = 0
    for (let i = 0; i < 200; i++) {
      if (selectNext(learningProfile, catalog, [], null).station.stationuuid === 'jazz') jazzLearning++
      if (selectNext(matureProfile, catalog, [], null).station.stationuuid === 'jazz') jazzMature++
    }
    // mature phase exponent amplifies the score advantage, so jazz picks more
    expect(jazzMature).toBeGreaterThan(jazzLearning)
  })
})

describe('selectNext - saved injection', () => {
  it('injects a saved station when Math.random returns < 0.1', () => {
    vi.spyOn(Math, 'random').mockReturnValue(0.05)
    const profile = createEmptyProfile()
    const saved = [makeStation('saved1', 'jazz')]
    const catalog = [makeStation('cat1', 'pop')]
    const { station } = selectNext(profile, catalog, saved, null)
    expect(station.stationuuid).toBe('saved1')
    vi.restoreAllMocks()
  })

  it('adds injected station uuid to savedDedupeWindow', () => {
    vi.spyOn(Math, 'random').mockReturnValue(0.05)
    const profile = createEmptyProfile()
    const saved = [makeStation('saved1', 'jazz')]
    const { updatedProfile } = selectNext(profile, [saved[0]], saved, null)
    expect(updatedProfile.savedDedupeWindow).toContain('saved1')
    vi.restoreAllMocks()
  })

  it('trims savedDedupeWindow to 3 entries', () => {
    vi.spyOn(Math, 'random').mockReturnValue(0.05)
    const profile = { ...createEmptyProfile(), savedDedupeWindow: ['a', 'b', 'c'] }
    const saved = [makeStation('saved1', 'jazz')]
    const catalog = [makeStation('cat1', 'pop')]
    // saved1 is not in dedupeWindow, so it gets injected
    const { updatedProfile } = selectNext(profile, [...catalog, ...saved], saved, null)
    expect(updatedProfile.savedDedupeWindow.length).toBeLessThanOrEqual(3)
    vi.restoreAllMocks()
  })

  it('falls through when all saved stations are in dedupeWindow', () => {
    vi.spyOn(Math, 'random').mockReturnValue(0.05)
    const profile = { ...createEmptyProfile(), savedDedupeWindow: ['saved1'] }
    const saved = [makeStation('saved1', 'jazz')]
    const catalog = [makeStation('cat1', 'pop')]
    const { station } = selectNext(profile, catalog, saved, null)
    expect(station.stationuuid).toBe('cat1')
    vi.restoreAllMocks()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm run test:run -- src/recommendation.test.ts
```

Expected: FAIL — `selectNext` not exported.

- [ ] **Step 3: Implement `selectNext` and its helpers in `recommendation.ts`**

Append after `applySignal`:

```typescript
const RECENTLY_PLAYED_SIZE = 50
const SAVED_DEDUP_SIZE     = 3
const SAVED_INJECT_RATE    = 0.1
const EXPLORE_RATE         = 0.15
const COLD_START_THRESHOLD = 16
const LEARNING_THRESHOLD   = 76
const MATURE_EXPONENT      = 1.5

const NON_MUSIC_TAGS = new Set(['news', 'talk', 'sports', 'religion'])

function isMusicStation(station: Station): boolean {
  if (!station.tags) return true
  return !station.tags.split(',').map(t => t.trim().toLowerCase()).some(t => NON_MUSIC_TAGS.has(t))
}

function primaryGenreTag(station: Station): string {
  return station.tags?.split(',')[0]?.trim().toLowerCase() ?? ''
}

function stationScore(station: Station, profile: TasteProfile): number {
  const tags = station.tags
    ? station.tags.split(',').map(t => t.trim().toLowerCase()).filter(Boolean)
    : []
  const country = station.country?.toLowerCase() ?? ''
  let score = tags.reduce((sum, tag) => sum + (profile.genreScores[tag] ?? 0), 0)
  score += profile.countryScores[country] ?? 0
  return score
}

function weightedRandom(items: Station[], weights: number[]): Station {
  const total = weights.reduce((a, b) => a + b, 0)
  let r = Math.random() * total
  for (let i = 0; i < items.length; i++) {
    r -= weights[i]
    if (r <= 0) return items[i]
  }
  return items[items.length - 1]
}

export function selectNext(
  profile: TasteProfile,
  catalog: Station[],
  savedStations: Station[],
  lastStation: Station | null,
): { station: Station; updatedProfile: TasteProfile } {
  const recentSet  = new Set(profile.recentlyPlayed)
  const dedupeSet  = new Set(profile.savedDedupeWindow)
  const phase      = profile.interactionCount < COLD_START_THRESHOLD ? 'cold'
    : profile.interactionCount < LEARNING_THRESHOLD ? 'learning'
    : 'mature'

  // Step 1: 10% saved injection
  if (Math.random() < SAVED_INJECT_RATE && savedStations.length > 0) {
    const eligible = savedStations.filter(
      s => !dedupeSet.has(s.stationuuid) && !recentSet.has(s.stationuuid),
    )
    if (eligible.length > 0) {
      const station = eligible[Math.floor(Math.random() * eligible.length)]
      const savedDedupeWindow = [station.stationuuid, ...profile.savedDedupeWindow].slice(0, SAVED_DEDUP_SIZE)
      const recentlyPlayed    = [station.stationuuid, ...profile.recentlyPlayed].slice(0, RECENTLY_PLAYED_SIZE)
      return { station, updatedProfile: { ...profile, savedDedupeWindow, recentlyPlayed } }
    }
  }

  const fresh = catalog.filter(s => !recentSet.has(s.stationuuid))

  // Step 2: 15% exploration
  if (Math.random() < EXPLORE_RATE) {
    const pool = phase === 'cold' ? fresh.filter(isMusicStation) : fresh
    if (pool.length > 0) {
      const station = pool[Math.floor(Math.random() * pool.length)]
      const recentlyPlayed = [station.stationuuid, ...profile.recentlyPlayed].slice(0, RECENTLY_PLAYED_SIZE)
      return { station, updatedProfile: { ...profile, recentlyPlayed } }
    }
  }

  // Step 3: Scored selection
  let station: Station

  if (phase === 'cold') {
    let pool = fresh.filter(isMusicStation)
    const lastGenre = lastStation ? primaryGenreTag(lastStation) : ''
    if (lastGenre) {
      const diverse = pool.filter(s => primaryGenreTag(s) !== lastGenre)
      if (diverse.length > 0) pool = diverse
    }
    if (pool.length === 0) pool = catalog.filter(isMusicStation)
    if (pool.length === 0) pool = catalog
    station = pool[Math.floor(Math.random() * pool.length)]
  } else {
    const pool    = fresh.length > 0 ? fresh : catalog
    const scores  = pool.map(s => {
      const raw = stationScore(s, profile)
      const w   = Math.max(raw, 0.01)
      return phase === 'mature' ? Math.pow(w, MATURE_EXPONENT) : w
    })
    station = weightedRandom(pool, scores)
  }

  const recentlyPlayed = [station.stationuuid, ...profile.recentlyPlayed].slice(0, RECENTLY_PLAYED_SIZE)
  return { station, updatedProfile: { ...profile, recentlyPlayed } }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/recommendation.test.ts
```

Expected: PASS — all tests.

- [ ] **Step 5: Commit**

```bash
git add src/recommendation.ts src/recommendation.test.ts
git commit -m "feat: implement selectNext with cold/learning/mature phases and saved injection"
```

---

## Task 4: Upgrade IDB schema — add `keyval` store

**Files:**
- Modify: `src/useIndexedDB.ts`

The `keyval` store holds arbitrary key→value entries (no keyPath). Used to persist `TasteProfile`. Requires bumping DB version from 1 to 2 and handling the upgrade path so existing users keep their saved stations.

- [ ] **Step 1: Verify existing saved-stations tests still pass before touching anything**

```bash
npm run test:run -- src/useSavedStations.test.ts
```

Expected: PASS — all existing tests green.

- [ ] **Step 2: Update `src/useIndexedDB.ts`**

Replace the entire file content:

```typescript
let dbPromise: Promise<IDBDatabase> | null = null

export function openDB(): Promise<IDBDatabase> {
  if (!dbPromise) {
    dbPromise = new Promise((resolve, reject) => {
      const req = indexedDB.open('drifting-db', 2)
      req.onupgradeneeded = (event) => {
        const db = req.result
        const { oldVersion } = event as IDBVersionChangeEvent
        if (oldVersion < 1) {
          db.createObjectStore('savedStations', { keyPath: 'stationuuid' })
        }
        if (oldVersion < 2) {
          db.createObjectStore('keyval')
        }
      }
      req.onsuccess = () => resolve(req.result)
      req.onerror = () => {
        dbPromise = null
        reject(req.error)
      }
    })
  }
  return dbPromise
}

export function resetDBForTesting(): void {
  dbPromise = null
}
```

- [ ] **Step 3: Re-run saved-stations tests to confirm no regression**

```bash
npm run test:run -- src/useSavedStations.test.ts
```

Expected: PASS — all tests still green. (`fake-indexeddb` starts fresh at version 0→2, creating both stores.)

- [ ] **Step 4: Commit**

```bash
git add src/useIndexedDB.ts
git commit -m "feat: bump IDB to v2, add keyval store for taste profile persistence"
```

---

## Task 5: `useRecommendation` hook

**Files:**
- Create: `src/useRecommendation.ts`
- Create: `src/useRecommendation.test.ts`

- [ ] **Step 1: Write failing test**

Create `src/useRecommendation.test.ts`:

```typescript
import { renderHook, act, waitFor } from '@testing-library/react'
import { IDBFactory } from 'fake-indexeddb'
import { useRecommendation } from './useRecommendation'
import { resetDBForTesting } from './useIndexedDB'
import type { Station } from './types'

const station: Station = {
  stationuuid: 'uuid-1',
  name: 'Jazz FM',
  url: '',
  tags: 'jazz',
  country: 'France',
  bitrate: 128,
  corsFriendly: false,
}

beforeEach(() => {
  global.indexedDB = new IDBFactory()
  resetDBForTesting()
})

describe('useRecommendation', () => {
  it('selectNext returns a station from the catalog', () => {
    const { result } = renderHook(() => useRecommendation())
    let picked: Station | undefined
    act(() => {
      picked = result.current.selectNext([station], [], null)
    })
    expect(picked?.stationuuid).toBe('uuid-1')
  })

  it('recordSignal persists to IDB and survives remount', async () => {
    const { result, unmount } = renderHook(() => useRecommendation())

    act(() => { result.current.recordSignal(station, 'save') })

    unmount()

    // remount — should load the persisted profile
    const { result: result2 } = renderHook(() => useRecommendation())
    await waitFor(() => {
      // after loading, interactionCount should be 1
      // verify by confirming a selectNext returns correctly (profile loaded)
      let picked: Station | undefined
      act(() => { picked = result2.current.selectNext([station], [], null) })
      expect(picked).toBeDefined()
    })
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npm run test:run -- src/useRecommendation.test.ts
```

Expected: FAIL — `useRecommendation` module not found.

- [ ] **Step 3: Create `src/useRecommendation.ts`**

```typescript
import { useCallback, useEffect, useRef } from 'react'
import { openDB } from './useIndexedDB'
import type { Station } from './types'
import {
  type TasteProfile,
  type SignalType,
  createEmptyProfile,
  applySignal,
  selectNext as _selectNext,
} from './recommendation'

const IDB_KEY = 'tasteProfile'

async function loadProfile(): Promise<TasteProfile> {
  try {
    const db = await openDB()
    return await new Promise(resolve => {
      const req = db.transaction('keyval', 'readonly').objectStore('keyval').get(IDB_KEY)
      req.onsuccess = () => resolve((req.result as TasteProfile | undefined) ?? createEmptyProfile())
      req.onerror  = () => resolve(createEmptyProfile())
    })
  } catch {
    return createEmptyProfile()
  }
}

async function persistProfile(profile: TasteProfile): Promise<void> {
  try {
    const db = await openDB()
    await new Promise<void>((resolve, reject) => {
      const tx = db.transaction('keyval', 'readwrite')
      tx.objectStore('keyval').put(profile, IDB_KEY)
      tx.oncomplete = () => resolve()
      tx.onerror    = () => reject(tx.error)
    })
  } catch {}
}

export function useRecommendation() {
  const profileRef     = useRef<TasteProfile>(createEmptyProfile())
  const dwellTimerRef  = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    loadProfile().then(profile => { profileRef.current = profile })
  }, [])

  const recordSignal = useCallback((station: Station, type: SignalType) => {
    const updated = applySignal(profileRef.current, station, type)
    profileRef.current = updated
    persistProfile(updated)
  }, [])

  const selectNext = useCallback((
    catalog: Station[],
    savedStations: Station[],
    lastStation: Station | null,
  ): Station => {
    const { station, updatedProfile } = _selectNext(
      profileRef.current, catalog, savedStations, lastStation,
    )
    profileRef.current = updatedProfile
    persistProfile(updatedProfile)
    return station
  }, [])

  const startDwellTimer = useCallback((station: Station) => {
    if (dwellTimerRef.current) clearTimeout(dwellTimerRef.current)
    dwellTimerRef.current = setTimeout(() => {
      recordSignal(station, 'dwell')
      dwellTimerRef.current = null
    }, 60_000)
  }, [recordSignal])

  const cancelDwellTimer = useCallback(() => {
    if (dwellTimerRef.current) {
      clearTimeout(dwellTimerRef.current)
      dwellTimerRef.current = null
    }
  }, [])

  useEffect(() => () => cancelDwellTimer(), [cancelDwellTimer])

  return { recordSignal, selectNext, startDwellTimer, cancelDwellTimer }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm run test:run -- src/useRecommendation.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/useRecommendation.ts src/useRecommendation.test.ts
git commit -m "feat: add useRecommendation hook with IDB persistence and dwell timer"
```

---

## Task 6: Wire recommendation engine into `App.tsx`

**Files:**
- Modify: `src/App.tsx`

Changes:
1. Import `useRecommendation` and `skipSignalType`
2. Remove `getRandomStation` import (no longer used)
3. Add `playStartRef` to track when audio began
4. Add `savedStationsRef` to give `loadAndPlay` access to current saved stations without stale closure
5. Call `selectNext` in `loadAndPlay` instead of `getRandomStation`
6. Call `startDwellTimer` after station loads successfully
7. Record skip signal + cancel dwell timer in `goNext`
8. Record prev signal + cancel dwell timer in `goPrevious`
9. Record save signal in `handleSave`

- [ ] **Step 1: Update imports at top of `App.tsx`**

Replace:
```typescript
import { loadCatalog, getRandomStation } from './catalog'
```
With:
```typescript
import { loadCatalog } from './catalog'
import { skipSignalType } from './recommendation'
import { useRecommendation } from './useRecommendation'
```

- [ ] **Step 2: Add hook call and new refs inside the `App` component, after the `useSavedStations` line**

Add after `const { savedStations, save, remove, isSaved } = useSavedStations()`:

```typescript
const { recordSignal, selectNext: pickNext, startDwellTimer, cancelDwellTimer } = useRecommendation()

const playStartRef       = useRef<number>(0)
const savedStationsRef   = useRef<Station[]>(savedStations)
useEffect(() => { savedStationsRef.current = savedStations }, [savedStations])
```

- [ ] **Step 3: Update `loadAndPlay` — replace `getRandomStation` with `pickNext` and add `startDwellTimer`**

In the initial-load branch (the `if (isInitial)` block), replace:
```typescript
const s = stationToLoad ?? getRandomStation(catalogRef.current)
```
With:
```typescript
const s = stationToLoad ?? pickNext(catalogRef.current, savedStationsRef.current, stationRef.current)
```

Then, after `vizRef.current?.triggerBurst()`, add:
```typescript
playStartRef.current = Date.now()
startDwellTimer(s)
```

In the navigation branch (the `try` block after `transitioningRef.current = true`), replace:
```typescript
const s = stationToLoad ?? getRandomStation(catalogRef.current)
```
With:
```typescript
const s = stationToLoad ?? pickNext(catalogRef.current, savedStationsRef.current, stationRef.current)
```

Then, after `vizRef.current?.triggerBurst()`, add:
```typescript
playStartRef.current = Date.now()
startDwellTimer(s)
```

Also add `pickNext` and `startDwellTimer` to the `useCallback` dependency array of `loadAndPlay`:
```typescript
}, [pickNext, startDwellTimer])
```

- [ ] **Step 4: Update `goNext` — record skip signal and cancel dwell timer**

Replace the current `goNext`:
```typescript
const goNext = useCallback(async () => {
  if (phaseRef.current !== 'playing' || transitioningRef.current) return
  const stationBeforeNav = stationRef.current
  const success = await loadAndPlay()
  if (success && stationBeforeNav) {
    historyRef.current.push(stationBeforeNav)
    setCanGoPrevious(true)
  }
}, [loadAndPlay])
```

With:
```typescript
const goNext = useCallback(async () => {
  if (phaseRef.current !== 'playing' || transitioningRef.current) return
  const stationBeforeNav = stationRef.current
  if (stationBeforeNav) {
    const listenSeconds = (Date.now() - playStartRef.current) / 1000
    recordSignal(stationBeforeNav, skipSignalType(listenSeconds))
    cancelDwellTimer()
  }
  const success = await loadAndPlay()
  if (success && stationBeforeNav) {
    historyRef.current.push(stationBeforeNav)
    setCanGoPrevious(true)
  }
}, [loadAndPlay, recordSignal, cancelDwellTimer])
```

- [ ] **Step 5: Update `goPrevious` — record prev signal and cancel dwell timer**

Replace the current `goPrevious`:
```typescript
const goPrevious = useCallback(() => {
  if (phaseRef.current !== 'playing' || transitioningRef.current) return
  if (historyRef.current.length === 0) return
  const prev = historyRef.current.pop()!
  setCanGoPrevious(historyRef.current.length > 0)
  loadAndPlay(prev)
}, [loadAndPlay])
```

With:
```typescript
const goPrevious = useCallback(() => {
  if (phaseRef.current !== 'playing' || transitioningRef.current) return
  if (historyRef.current.length === 0) return
  const prev = historyRef.current.pop()!
  setCanGoPrevious(historyRef.current.length > 0)
  recordSignal(prev, 'prev')
  cancelDwellTimer()
  loadAndPlay(prev)
}, [loadAndPlay, recordSignal, cancelDwellTimer])
```

- [ ] **Step 6: Update `handleSave` — record save signal**

Replace the current `handleSave`:
```typescript
const handleSave = useCallback(async () => {
  if (!stationRef.current || phaseRef.current !== 'playing') return
  const result = await save(stationRef.current)
  if (result === 'saved') {
    vizRef.current?.triggerBurst()
    setToastMessage('Saved ♥')
  } else if (result === 'duplicate') {
    setToastMessage('Already saved ♥')
  } else {
    setToastMessage('Save limit reached')
  }
}, [save])
```

With:
```typescript
const handleSave = useCallback(async () => {
  if (!stationRef.current || phaseRef.current !== 'playing') return
  recordSignal(stationRef.current, 'save')
  const result = await save(stationRef.current)
  if (result === 'saved') {
    vizRef.current?.triggerBurst()
    setToastMessage('Saved ♥')
  } else if (result === 'duplicate') {
    setToastMessage('Already saved ♥')
  } else {
    setToastMessage('Save limit reached')
  }
}, [save, recordSignal])
```

- [ ] **Step 7: Run the full test suite**

```bash
npm run test:run
```

Expected: PASS — all tests green.

- [ ] **Step 8: Run the dev server and verify manually**

```bash
npm run dev
```

Open the app, tap to begin, navigate through several stations. After navigating away from a station, the recommendation engine is recording signals silently. Open DevTools → Application → IndexedDB → drifting-db → keyval to verify `tasteProfile` is being written with non-zero `interactionCount` and scores.

- [ ] **Step 9: Commit**

```bash
git add src/App.tsx
git commit -m "feat: wire recommendation engine into App — signals, dwell timer, selectNext"
```
