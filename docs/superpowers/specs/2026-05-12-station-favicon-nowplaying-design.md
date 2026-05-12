# Station Favicon + Now-Playing Design

**Date:** 2026-05-12  
**Status:** Approved

## Goal

Add two new elements to the station info overlay:

1. **Favicon tile** — 32×32px rounded square to the left of the station name/country block, showing a genre-seeded radial gradient immediately and fading in the station's favicon on top if available.
2. **Now-playing line** — artist–track string from ICY stream metadata, appearing above the station name when available.

Station name, country, and favicon never shift position. Track title and genre appear/disappear without disturbing the anchored block.

---

## Data Model

### `src/types.ts`

Add optional field to `Station`:

```typescript
favicon_url?: string
```

### `scripts/buildCatalog.ts`

Add `favicon_url?: string` to `RawStation` and `CatalogStation` interfaces. Capture the `favicon` field from the radio-browser.info API response and write it into the catalog entry. Rebuild `public/catalog.json` after the script change.

---

## Layout

The station info block is anchored at `position: fixed; bottom: 40px; left: 40px`. Track title and genre are `position: absolute` relative to the outer container so they don't affect its height — name/country/favicon never move.

```
outer div (position: fixed, bottom: 40, left: 40, pointer-events: none)
  track-title div (position: absolute, bottom: 100%, left: 44px)  ← above, conditional
  main-row div (display: flex, align-items: flex-start, gap: 12)
    favicon tile (32×32px)
    text-col div
      name + ♥ badge
      country
  genre div (position: absolute, top: 100%, left: 44px)           ← below, conditional
```

`44px = favicon (32) + gap (12)` — track title and genre align with the station name's left edge.

---

## Favicon Tile

- **Size:** 32×32px, `border-radius: 8px`
- **Background:** radial gradient from `getGenreColor(station.tags)` (bright center → dark edge), rendered immediately
- **Image:** `<img src={station.favicon_url}>`, `opacity: 0` initially; transitions to `opacity: 1` on `onLoad`; stays hidden on `onError` (gradient shows through)
- No broken-image icon, no initials fallback — gradient is sufficient

---

## `useNowPlaying` Hook

**File:** `src/useNowPlaying.ts`

**Signature:**
```typescript
function useNowPlaying(station: Station | null): string | null
```

**Behavior:**

- Returns `null` when station is null or `!station.corsFriendly`
- On each station change, aborts any in-progress fetch via `AbortController`
- Fetches the stream with `Icy-MetaData: 1` header
- Reads `Icy-MetaInt` from response headers — if absent, stream doesn't support ICY; returns null
- Buffers incoming bytes until it has enough to reach the first metadata block:
  - Skip `metaInt` bytes of audio data
  - Read 1 length byte; multiply by 16 to get metadata block size
  - Read metadata block; cancel the stream reader
- Parses `StreamTitle='...';` via regex; sets `nowPlaying` state. Empty `StreamTitle` (`''`) is treated as null — no track line shown
- Polls every 30s to catch track changes while the station is playing
- Cleans up (abort + clear poll interval) on unmount or station change

**ICY metadata parsing:**
```
response headers → Icy-MetaInt: N
stream bytes:  [N audio bytes][1 length byte][length*16 metadata bytes]...
metadata text: StreamTitle='Artist - Title';StreamUrl='...';
```

---

## `StationInfo` Component

**File:** `src/StationInfo.tsx`  
**Test file:** `src/StationInfo.test.tsx`

**Props:**
```typescript
interface Props {
  station: Station
  isSaved: boolean
  nowPlaying: string | null
}
```

Renders the layout described above. All existing text styles preserved:
- Track title: 13px, weight 400, `rgba(255,255,255,0.55)`, `letterSpacing: 0.02em`
- Name: 16px, weight 500, `rgba(255,255,255,0.85)` (or 0.95 when saved)
- ♥ badge: 12px, `rgba(255,255,255,0.5)`, `marginLeft: 6`
- Country: 13px, weight 400, `rgba(255,255,255,0.45)`, `letterSpacing: 0.04em`
- Genre: 13px, weight 400, `rgba(255,255,255,0.35)`, `letterSpacing: 0.04em`

---

## `App.tsx` Changes

- Remove the inline station info JSX block
- Add `const nowPlaying = useNowPlaying(station)`
- Render `<StationInfo station={station} isSaved={isSaved(station.stationuuid)} nowPlaying={nowPlaying} />`

---

## File Map

| File | Action |
|---|---|
| `src/types.ts` | Add `favicon_url?: string` to `Station` |
| `scripts/buildCatalog.ts` | Add `favicon_url` to `RawStation` + `CatalogStation`; capture from API |
| `public/catalog.json` | Rebuild after script change |
| `src/useNowPlaying.ts` | Create — ICY metadata hook |
| `src/useNowPlaying.test.ts` | Create — unit tests with mocked fetch |
| `src/StationInfo.tsx` | Create — station info overlay component |
| `src/StationInfo.test.tsx` | Create — render + prop tests |
| `src/App.tsx` | Remove inline station block; wire `useNowPlaying` + `StationInfo` |

---

## Testing

- `useNowPlaying`: mock `fetch` to return controlled headers and byte streams; assert correct `nowPlaying` values for CORS/non-CORS stations, missing `Icy-MetaInt`, and polling behavior
- `StationInfo`: render tests for all conditional states (no track title, with track title, no genre, with genre, saved/unsaved, favicon load/error)
