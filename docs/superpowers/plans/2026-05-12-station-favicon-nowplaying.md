# Station Favicon + Now-Playing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a 32×32 genre-colored favicon tile to the left of the station info block, and show the current ICY stream track title above the station name when available.

**Architecture:** Four tasks in order: (1) add `favicon_url` to the data model and rebuild the catalog, (2) create `useNowPlaying` hook with ICY stream parsing, (3) create `StationInfo` component that renders the new layout, (4) wire both into `App.tsx` and remove the old inline block.

**Tech Stack:** React 19, TypeScript, Vitest + Testing Library, inline styles, `getGenreColor` from existing `src/visualization/genreColors.ts`

---

## File Map

| File | Action |
|---|---|
| `src/types.ts` | Add `favicon_url?: string` to `Station` |
| `scripts/buildCatalog.ts` | Add `favicon` to `RawStation`; add `favicon_url` to `CatalogStation`; copy field in map |
| `public/catalog.json` | Rebuilt by running `npx tsx scripts/buildCatalog.ts --limit=50` |
| `src/useNowPlaying.ts` | Create — ICY metadata fetch + parse hook |
| `src/useNowPlaying.test.ts` | Create — unit tests with mocked fetch |
| `src/StationInfo.tsx` | Create — station info overlay component |
| `src/StationInfo.test.tsx` | Create — render tests for all conditional states |
| `src/App.tsx` | Import + wire `useNowPlaying` and `StationInfo`; remove inline station info block |

---

## Task 1: Data model + catalog rebuild

**Files:**
- Modify: `src/types.ts`
- Modify: `scripts/buildCatalog.ts`
- Rebuild: `public/catalog.json`

- [ ] **Step 1: Add `favicon_url` to `Station` in `src/types.ts`**

Replace the existing `Station` interface:

```typescript
export interface Station {
  stationuuid: string
  name: string
  url: string
  tags: string
  country: string
  bitrate: number
  corsFriendly: boolean
  favicon_url?: string
}
```

- [ ] **Step 2: Add `favicon` to `RawStation` and `favicon_url` to `CatalogStation` in `scripts/buildCatalog.ts`**

Replace the `RawStation` interface:

```typescript
interface RawStation {
  stationuuid: string
  name: string
  url: string
  url_resolved: string
  tags: string
  country: string
  bitrate: number
  lastcheckok: number
  favicon: string
}
```

Replace the `CatalogStation` interface:

```typescript
export interface CatalogStation {
  stationuuid: string
  name: string
  url: string
  tags: string
  country: string
  bitrate: number
  corsFriendly: boolean
  favicon_url?: string
}
```

- [ ] **Step 3: Copy `favicon` into the catalog in the `main()` map**

Replace the `catalog` map in `scripts/buildCatalog.ts`:

```typescript
  const catalog: CatalogStation[] = stations.map((s, i) => ({
    stationuuid: s.stationuuid,
    name: s.name,
    url: s.url_resolved || s.url,
    tags: s.tags,
    country: s.country,
    bitrate: s.bitrate,
    corsFriendly: corsResults[i],
    ...(s.favicon ? { favicon_url: s.favicon } : {}),
  }))
```

- [ ] **Step 4: Run tests to confirm existing tests still pass**

```bash
npm run test:run
```

Expected: all tests pass (no type errors, `buildCatalog.test.ts` still passes).

- [ ] **Step 5: Rebuild the catalog**

```bash
npx tsx scripts/buildCatalog.ts --limit=50
```

Expected: output ends with `Written to public/catalog.json`. Open `public/catalog.json` and spot-check that some entries now have a `favicon_url` field.

- [ ] **Step 6: Commit**

```bash
git add src/types.ts scripts/buildCatalog.ts public/catalog.json
git commit -m "feat: add favicon_url to Station type and rebuild catalog"
```

---

## Task 2: `useNowPlaying` hook

**Files:**
- Create: `src/useNowPlaying.ts`
- Create: `src/useNowPlaying.test.ts`

- [ ] **Step 1: Create the test file**

Create `src/useNowPlaying.test.ts`:

```typescript
import { renderHook, waitFor } from '@testing-library/react'
import { vi, describe, it, expect, beforeEach } from 'vitest'
import { useNowPlaying } from './useNowPlaying'
import type { Station } from './types'

const corsStation: Station = {
  stationuuid: 'uuid-1',
  name: 'Test FM',
  url: 'http://test.fm/stream',
  tags: 'jazz',
  country: 'Germany',
  bitrate: 128,
  corsFriendly: true,
}

const nonCorsStation: Station = { ...corsStation, corsFriendly: false }

function makeIcyFetch(metaInt: number, streamTitle: string) {
  const metaContent = `StreamTitle='${streamTitle}';`
  const encoder = new TextEncoder()
  const metaBytes = encoder.encode(metaContent)
  const metaLenByte = Math.ceil(metaBytes.length / 16)
  const metaPadded = new Uint8Array(metaLenByte * 16)
  metaPadded.set(metaBytes)

  const audio = new Uint8Array(metaInt)
  const full = new Uint8Array(metaInt + 1 + metaPadded.length)
  full.set(audio, 0)
  full[metaInt] = metaLenByte
  full.set(metaPadded, metaInt + 1)

  let delivered = false
  return {
    headers: { get: (name: string) => name === 'icy-metaint' ? String(metaInt) : null },
    body: {
      getReader: () => ({
        read: () => {
          if (!delivered) {
            delivered = true
            return Promise.resolve({ value: full, done: false })
          }
          return Promise.resolve({ value: undefined, done: true })
        },
        cancel: () => Promise.resolve(),
      }),
    },
  }
}

function makeNoIcyFetch() {
  return {
    headers: { get: () => null },
    body: { getReader: () => ({ read: () => Promise.resolve({ value: undefined, done: true }), cancel: () => Promise.resolve() }) },
  }
}

beforeEach(() => { vi.restoreAllMocks() })

describe('useNowPlaying', () => {
  it('returns null immediately for null station', () => {
    const { result } = renderHook(() => useNowPlaying(null))
    expect(result.current).toBeNull()
  })

  it('returns null immediately for non-CORS station without calling fetch', () => {
    const spy = vi.spyOn(globalThis, 'fetch')
    const { result } = renderHook(() => useNowPlaying(nonCorsStation))
    expect(result.current).toBeNull()
    expect(spy).not.toHaveBeenCalled()
  })

  it('returns null when response has no icy-metaint header', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(makeNoIcyFetch() as unknown as Response)
    const { result } = renderHook(() => useNowPlaying(corsStation))
    await waitFor(() => expect(globalThis.fetch).toHaveBeenCalledTimes(1))
    expect(result.current).toBeNull()
  })

  it('parses StreamTitle from ICY stream', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(makeIcyFetch(8192, 'Miles Davis - So What') as unknown as Response)
    const { result } = renderHook(() => useNowPlaying(corsStation))
    await waitFor(() => expect(result.current).toBe('Miles Davis - So What'))
  })

  it('returns null for empty StreamTitle', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(makeIcyFetch(8192, '') as unknown as Response)
    const { result } = renderHook(() => useNowPlaying(corsStation))
    await waitFor(() => expect(globalThis.fetch).toHaveBeenCalledTimes(1))
    expect(result.current).toBeNull()
  })

  it('resets to null when station changes to null', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(makeIcyFetch(8192, 'Artist - Track') as unknown as Response)
    const { result, rerender } = renderHook(
      ({ s }: { s: Station | null }) => useNowPlaying(s),
      { initialProps: { s: corsStation } },
    )
    await waitFor(() => expect(result.current).toBe('Artist - Track'))
    rerender({ s: null })
    await waitFor(() => expect(result.current).toBeNull())
  })
})
```

- [ ] **Step 2: Run to confirm they fail**

```bash
npm run test:run -- src/useNowPlaying.test.ts
```

Expected: FAIL — `Cannot find module './useNowPlaying'`

- [ ] **Step 3: Create `src/useNowPlaying.ts`**

```typescript
import { useState, useEffect, useRef } from 'react'
import type { Station } from './types'

async function fetchNowPlaying(url: string, signal: AbortSignal): Promise<string | null> {
  const res = await fetch(url, { headers: { 'Icy-MetaData': '1' }, signal })

  const metaIntStr = res.headers.get('icy-metaint')
  if (!metaIntStr) return null
  const metaInt = parseInt(metaIntStr, 10)
  if (isNaN(metaInt) || metaInt <= 0) return null

  if (!res.body) return null
  const reader = res.body.getReader()
  let buf = new Uint8Array(0)

  try {
    while (true) {
      const { value, done } = await reader.read()
      if (done) break
      if (value) {
        const next = new Uint8Array(buf.length + value.length)
        next.set(buf)
        next.set(value, buf.length)
        buf = next
      }
      if (buf.length < metaInt + 1) continue
      const metaLenByte = buf[metaInt]
      const metaLen = metaLenByte * 16
      if (metaLen === 0) return null
      if (buf.length < metaInt + 1 + metaLen) continue
      const metaBytes = buf.slice(metaInt + 1, metaInt + 1 + metaLen)
      const metaStr = new TextDecoder().decode(metaBytes)
      const match = metaStr.match(/StreamTitle='([^']*)';/)
      return match?.[1] || null
    }
  } finally {
    void reader.cancel()
  }
  return null
}

export function useNowPlaying(station: Station | null): string | null {
  const [nowPlaying, setNowPlaying] = useState<string | null>(null)
  const abortRef = useRef<AbortController | null>(null)
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null)

  useEffect(() => {
    abortRef.current?.abort()
    if (intervalRef.current) clearInterval(intervalRef.current)
    setNowPlaying(null)

    if (!station || !station.corsFriendly) return

    const poll = async () => {
      const controller = new AbortController()
      abortRef.current = controller
      try {
        const title = await fetchNowPlaying(station.url, controller.signal)
        if (!controller.signal.aborted) setNowPlaying(title)
      } catch {
        // fetch errors and aborts — silently ignore
      }
    }

    poll()
    intervalRef.current = setInterval(poll, 30_000)

    return () => {
      abortRef.current?.abort()
      if (intervalRef.current) clearInterval(intervalRef.current)
    }
  }, [station])

  return nowPlaying
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/useNowPlaying.test.ts
```

Expected: all 6 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/useNowPlaying.ts src/useNowPlaying.test.ts
git commit -m "feat: add useNowPlaying hook with ICY stream metadata parsing"
```

---

## Task 3: `StationInfo` component

**Files:**
- Create: `src/StationInfo.tsx`
- Create: `src/StationInfo.test.tsx`

- [ ] **Step 1: Create the test file**

Create `src/StationInfo.test.tsx`:

```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { vi, describe, it, expect } from 'vitest'
import { StationInfo } from './StationInfo'
import type { Station } from './types'

const station: Station = {
  stationuuid: 'uuid-1',
  name: 'Jazz Radio',
  url: 'http://jazz.fm/stream',
  tags: 'jazz,soul',
  country: 'France',
  bitrate: 128,
  corsFriendly: true,
  favicon_url: 'http://jazz.fm/favicon.png',
}

describe('StationInfo', () => {
  it('renders station name', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    expect(screen.getByText('Jazz Radio')).toBeInTheDocument()
  })

  it('renders country', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    expect(screen.getByText('France')).toBeInTheDocument()
  })

  it('renders first genre tag when tags are present', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    expect(screen.getByText('Jazz')).toBeInTheDocument()
  })

  it('does not render genre when tags are empty', () => {
    render(<StationInfo station={{ ...station, tags: '' }} isSaved={false} nowPlaying={null} />)
    expect(screen.queryByText('Jazz')).not.toBeInTheDocument()
  })

  it('renders ♥ badge when saved', () => {
    render(<StationInfo station={station} isSaved={true} nowPlaying={null} />)
    expect(screen.getByText('♥')).toBeInTheDocument()
  })

  it('does not render ♥ badge when not saved', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    expect(screen.queryByText('♥')).not.toBeInTheDocument()
  })

  it('renders nowPlaying text when provided', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying="Miles Davis - So What" />)
    expect(screen.getByText('Miles Davis - So What')).toBeInTheDocument()
  })

  it('does not render nowPlaying line when null', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    expect(screen.queryByText('Miles Davis - So What')).not.toBeInTheDocument()
  })

  it('renders favicon img with correct src when favicon_url is present', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    expect(screen.getByRole('img')).toHaveAttribute('src', 'http://jazz.fm/favicon.png')
  })

  it('does not render img when favicon_url is absent', () => {
    render(<StationInfo station={{ ...station, favicon_url: undefined }} isSaved={false} nowPlaying={null} />)
    expect(screen.queryByRole('img')).not.toBeInTheDocument()
  })

  it('fades in favicon on load', () => {
    render(<StationInfo station={station} isSaved={false} nowPlaying={null} />)
    const img = screen.getByRole('img')
    expect(img).toHaveStyle({ opacity: '0' })
    fireEvent.load(img)
    expect(img).toHaveStyle({ opacity: '1' })
  })
})
```

- [ ] **Step 2: Run to confirm they fail**

```bash
npm run test:run -- src/StationInfo.test.tsx
```

Expected: FAIL — `Cannot find module './StationInfo'`

- [ ] **Step 3: Create `src/StationInfo.tsx`**

```typescript
import { useState } from 'react'
import { getGenreColor } from './visualization/genreColors'
import { normalizeCountry, primaryGenre } from './utils'
import type { Station } from './types'

interface Props {
  station: Station
  isSaved: boolean
  nowPlaying: string | null
}

export function StationInfo({ station, isSaved, nowPlaying }: Props) {
  const [faviconLoaded, setFaviconLoaded] = useState(false)

  const c = getGenreColor(station.tags)
  const r = Math.round(c.r * 255)
  const g = Math.round(c.g * 255)
  const b = Math.round(c.b * 255)

  return (
    <div style={{
      position: 'fixed',
      bottom: 40,
      left: 40,
      pointerEvents: 'none',
      textShadow: '0 1px 12px rgba(0,0,0,1)',
    }}>
      {nowPlaying && (
        <div style={{
          position: 'absolute',
          bottom: '100%',
          left: 44,
          paddingBottom: 6,
          fontFamily: 'system-ui, sans-serif',
          fontSize: 13,
          fontWeight: 400,
          color: 'rgba(255,255,255,0.55)',
          letterSpacing: '0.02em',
          whiteSpace: 'nowrap',
        }}>
          {nowPlaying}
        </div>
      )}

      <div style={{ display: 'flex', alignItems: 'flex-start', gap: 12 }}>
        <div style={{
          width: 32,
          height: 32,
          borderRadius: 8,
          flexShrink: 0,
          background: `radial-gradient(circle at center, rgb(${r},${g},${b}) 0%, rgba(${r},${g},${b},0.15) 100%)`,
          overflow: 'hidden',
          position: 'relative',
        }}>
          {station.favicon_url && (
            <img
              src={station.favicon_url}
              alt=""
              onLoad={() => setFaviconLoaded(true)}
              onError={() => setFaviconLoaded(false)}
              style={{
                position: 'absolute',
                inset: 0,
                width: '100%',
                height: '100%',
                objectFit: 'cover',
                opacity: faviconLoaded ? 1 : 0,
                transition: 'opacity 0.3s',
              }}
            />
          )}
        </div>

        <div>
          <div style={{
            fontFamily: 'system-ui, sans-serif',
            fontSize: 16,
            fontWeight: 500,
            color: isSaved ? 'rgba(255,255,255,0.95)' : 'rgba(255,255,255,0.85)',
            letterSpacing: '0.01em',
          }}>
            {station.name}
            {isSaved && (
              <span style={{ marginLeft: 6, fontSize: 12, color: 'rgba(255,255,255,0.5)' }}>♥</span>
            )}
          </div>
          <div style={{
            marginTop: 4,
            fontFamily: 'system-ui, sans-serif',
            fontSize: 13,
            fontWeight: 400,
            color: 'rgba(255,255,255,0.45)',
            letterSpacing: '0.04em',
          }}>
            {normalizeCountry(station.country)}
          </div>
        </div>
      </div>

      {station.tags && (
        <div style={{
          position: 'absolute',
          top: '100%',
          left: 44,
          paddingTop: 2,
          fontFamily: 'system-ui, sans-serif',
          fontSize: 13,
          fontWeight: 400,
          color: 'rgba(255,255,255,0.35)',
          letterSpacing: '0.04em',
        }}>
          {primaryGenre(station.tags)}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm run test:run -- src/StationInfo.test.tsx
```

Expected: all 11 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/StationInfo.tsx src/StationInfo.test.tsx
git commit -m "feat: add StationInfo component with favicon tile and now-playing slot"
```

---

## Task 4: Wire into `App.tsx`

**Files:**
- Modify: `src/App.tsx`

No new tests — `App.tsx` wiring is verified manually via dev server.

- [ ] **Step 1: Add imports to `src/App.tsx`**

Add these two lines alongside the existing imports at the top of `src/App.tsx`:

```typescript
import { useNowPlaying } from './useNowPlaying'
import { StationInfo } from './StationInfo'
```

- [ ] **Step 2: Add `useNowPlaying` call inside `App()`**

After the `useRecommendation` destructure line, add:

```typescript
const nowPlaying = useNowPlaying(station)
```

- [ ] **Step 3: Replace the inline station info block**

In `src/App.tsx`, remove this entire block (lines 293–334):

```typescript
      {phase === 'playing' && station && (
        <div style={{
          position: 'fixed', bottom: 40, left: 40,
          pointerEvents: 'none',
          textShadow: '0 1px 12px rgba(0,0,0,1)',
        }}>
          <div style={{
            fontFamily: 'system-ui, sans-serif',
            fontSize: 16,
            fontWeight: 500,
            color: isSaved(station.stationuuid) ? 'rgba(255,255,255,0.95)' : 'rgba(255,255,255,0.85)',
            letterSpacing: '0.01em',
          }}>
            {station.name}
            {isSaved(station.stationuuid) && (
              <span style={{ marginLeft: 6, fontSize: 12, color: 'rgba(255,255,255,0.5)' }}>♥</span>
            )}
          </div>
          <div style={{
            marginTop: 4,
            fontFamily: 'system-ui, sans-serif',
            fontSize: 13,
            fontWeight: 400,
            color: 'rgba(255,255,255,0.45)',
            letterSpacing: '0.04em',
          }}>
            {normalizeCountry(station.country)}
          </div>
          {station.tags && (
            <div style={{
              marginTop: 2,
              fontFamily: 'system-ui, sans-serif',
              fontSize: 13,
              fontWeight: 400,
              color: 'rgba(255,255,255,0.35)',
              letterSpacing: '0.04em',
            }}>
              {primaryGenre(station.tags)}
            </div>
          )}
        </div>
      )}
```

Replace it with:

```typescript
      {phase === 'playing' && station && (
        <StationInfo
          station={station}
          isSaved={isSaved(station.stationuuid)}
          nowPlaying={nowPlaying}
        />
      )}
```

- [ ] **Step 4: Run the full test suite**

```bash
npm run test:run
```

Before running, also remove the now-unused `utils` import from `src/App.tsx`. Find and delete this line:

```typescript
import { normalizeCountry, primaryGenre } from './utils'
```

Then run:

```bash
npm run test:run
```

Expected: all tests pass.

- [ ] **Step 5: Verify in the dev server**

```bash
npm run dev
```

Checklist:
- Tap to begin → station info appears bottom-left with a 32×32 colored tile to the left of the name
- Tile background matches the genre color (warm for jazz, blue for electronic, etc.)
- If station has a favicon URL, the image fades in over the gradient after it loads
- Station name, country stay in place as you navigate — only the tile color changes
- For a CORS-friendly station: wait ~5s — if the stream exposes ICY metadata, the track title appears above the station name aligned with the name's left edge
- Genre appears below country, aligned with station name left edge
- Navigate to a new station → tile color updates instantly, track title clears and re-fetches

- [ ] **Step 6: Commit**

```bash
git add src/App.tsx
git commit -m "feat: wire StationInfo and useNowPlaying into App"
```

---

## Post-implementation

Update Section 17 of `/Users/vitorfreitas/dev/drifting/docs/web-design-spec.md`:

Change:
```
| Station info overlay | ✅ Done — includes ♥ badge when station is saved |
```

To:
```
| Station info overlay | ✅ Done — favicon tile (genre gradient fallback), now-playing via ICY metadata (CORS stations only), ♥ badge; name/country/favicon anchored; track title and genre float above/below without shifting |
```
