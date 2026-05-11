# Recommendation Engine — Design Spec

**Date:** 2026-05-11
**Status:** Approved
**Repo:** drifting-web

---

## Overview

Client-side recommendation engine that silently builds a taste profile from listening signals and uses it to select stations. No backend, no account — all state in IndexedDB.

---

## Data Model

`TasteProfile` stored in IndexedDB under key `"tasteProfile"`:

```typescript
interface TasteProfile {
  interactionCount: number
  genreScores: Record<string, number>    // tag → weight, floored at -10
  countryScores: Record<string, number>  // country → weight, floored at -10
  recentlyPlayed: string[]               // stationuuid[], last 50, excluded from selection
  savedDedupeWindow: string[]            // last 3 saved stations injected, prevents consecutive repeats
}
```

Empty profile: all fields zeroed/empty, created on first use.

---

## Signal Weights

| Signal type | Condition | Weight |
|---|---|---|
| `skip_fast` | navigate next < 5s | -3 |
| `skip_mid` | navigate next 5–30s | -1.5 |
| `skip_slow` | navigate next > 30s | -0.5 |
| `dwell` | 60s without navigation gesture | +0.5 |
| `prev` | navigate previous (applied to station being returned *to*) | +1 |
| `save` | save gesture | +3 |

`applySignal(profile, station, type)` distributes the weight across all of the station's tags (comma-split, trimmed, lowercased) and its country. Genre/country scores are floored at -10. `interactionCount` increments on skip and prev and save — not on dwell.

Signal weighting only applies once audio has successfully started playing. Failed streams record no signal.

---

## Station Selection

`selectNext(profile, catalog, savedStations, lastStation)` runs this decision tree:

### Step 1 — Saved injection (10% chance)
- Pick a random saved station excluding `savedDedupeWindow` (last 3 injected) and `recentlyPlayed`
- If none qualify, fall through to step 2
- On success: prepend UUID to `savedDedupeWindow` (trim to 3)

### Step 2 — Exploration (15% chance of remaining)
- Cold start: random music-only station, exclude `recentlyPlayed`
- Learning/Mature: any station, exclude `recentlyPlayed`

### Step 3 — Scored selection (~76.5%)

**Phase determination:**
- Cold start: `interactionCount < 16`
- Learning: `16 ≤ interactionCount < 76`
- Mature: `interactionCount ≥ 76`

**Music-only definition:** stations whose tags do not include `news`, `talk`, `sports`, or `religion` (same filter as iOS spec).

**Cold start:** random pick from music-only stations, avoiding same primary genre as `lastStation`, excluding `recentlyPlayed`.

**Learning:** score each candidate = `sum(genreScores[tag] for tag in station.tags) + countryScores[station.country]`. For weighted random selection, station weight = `Math.max(score, 0.01)` — negative stored scores don't exclude a station, they just make it very unlikely. Weighted random pick excluding `recentlyPlayed`.

**Mature:** same as learning but `score = raw_score ^ 1.5` (amplifies exploitation of strong preferences).

After any selection: prepend chosen UUID to `recentlyPlayed` (trim to 50).

---

## Hook: `useRecommendation`

Wraps the pure functions and handles React lifecycle + IDB persistence.

**Exposes:**
- `selectNext(catalog, savedStations, lastStation) → Station`
- `recordSignal(station, listenSeconds, type) → void`
- `startDwellTimer(station) → void` — fires `recordSignal(station, 60, 'dwell')` after 60s
- `cancelDwellTimer() → void`

**IDB persistence:** loads profile on mount, persists after every `recordSignal` call.

---

## App.tsx Integration

**`playStartRef`** — `Date.now()` set when a station successfully starts playing.

**`goNext`:**
1. Compute `listenSeconds = (Date.now() - playStartRef) / 1000`
2. Call `recordSignal(currentStation, listenSeconds, skip_*)` — type derived from duration
3. Call `cancelDwellTimer()`
4. Navigate

**`goPrevious`:**
1. Call `recordSignal(prevStation, 0, 'prev')` — signal for station being returned to
2. Call `cancelDwellTimer()`
3. Navigate

**`handleSave`:** calls `recordSignal(currentStation, 0, 'save')` — does not cancel dwell timer.

**After successful station load:** calls `startDwellTimer(newStation)`.

**Station selection:** replace `getRandomStation(catalog)` with `selectNext(catalog, savedStations, lastStation)`.

---

## Files

| File | Change |
|---|---|
| `src/recommendation.ts` | New — pure functions: `createEmptyProfile`, `applySignal`, `selectNext` |
| `src/useRecommendation.ts` | New — hook: IDB persistence, dwell timer, exposes `recordSignal` + `selectNext` |
| `src/App.tsx` | Modified — `playStartRef`, signal recording on nav, dwell timer calls, replace `getRandomStation` |
| `src/recommendation.test.ts` | New — unit tests for pure functions |

---

## What's excluded (deferred)

- Genre seed on first launch (floating chip cards) — not in web spec
- Locale-aware cold start bias — not in web spec
- Tempo / energy level learning — not in web spec
- Time-of-day patterns — not in web spec
