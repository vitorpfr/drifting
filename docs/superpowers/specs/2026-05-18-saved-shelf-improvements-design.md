# Saved Shelf Improvements — Design Spec

**Date:** 2026-05-18  
**Status:** Approved  
**Repo:** spectrale-web

---

## Overview

Four improvements to the saved stations shelf: animated playing indicator, sort controls, drag-to-reorder, and keyboard shortcuts (F to open, 1–0 to play).

---

## 1. Playing Indicator

Replace the existing `▶` text symbol on the currently playing station with a small CSS animated equalizer — three bars that pulse via `@keyframes`. Classic "now playing" visual, fits the ambient aesthetic.

- Shown only on the row whose `stationuuid` matches the currently playing station
- Animation: bars stagger their heights (short → tall → medium → short) on a ~0.9s loop
- Color: `rgba(255,255,255,0.55)` — visible but not dominant

---

## 2. Sort Bar

A row of 4 pill chips rendered just below the drag handle, above the station list.

| Label | Sort key |
|---|---|
| Saved | Custom drag order (localStorage `shelf_order`) |
| A–Z | `station.name` ascending |
| Country | `normalizeCountry(station.country)` ascending |
| Genre | `primaryGenre(station.tags)` ascending |

- Active chip: `rgba(255,255,255,0.85)`, white border `rgba(255,255,255,0.2)`
- Inactive chip: `rgba(255,255,255,0.30)`, no border
- Sort preference persisted to `localStorage` key `shelf_sort` (`'saved' | 'az' | 'country' | 'genre'`), defaults to `'saved'`
- Sort state lifted out of `SavedShelf` into a small `useShelfSort` hook in `App.tsx` so both the shelf and `useNavigation` operate on the same sorted array

---

## 3. Drag to Reorder

Available only when sort = `'saved'`. Disabled (handles hidden) for other sort modes.

**Drag handle:** `⠿` character on the left of each row, `rgba(255,255,255,0.25)`, only rendered in Saved sort mode.

**Desktop (HTML5 DnD):**
- Row has `draggable={true}` in Saved mode
- `onDragStart` stores the dragged index
- `onDragOver` computes drop target index and reorders a local preview array (visual only until drop)
- `onDrop` commits the new order

**Mobile (touch events):**
- `onTouchStart` records the touch position and dragged index
- `onTouchMove` calls `document.elementFromPoint` at touch position to find the current hover target row
- `onTouchEnd` commits the new order
- A `data-index` attribute on each row makes target identification reliable

**Persistence:**
- Custom order stored in `localStorage` key `shelf_order` as `string[]` (UUID array)
- Updated on every drop
- New saves (from `useSavedStations.save()`) are appended to the end of the order array; `useSavedStations` calls a `appendToOrder(uuid)` helper exported from the sort hook
- Stations whose UUID is absent from the order array are appended at the end in insertion order (safe fallback for data from before this feature)

---

## 4. Keyboard Shortcuts

### F key — open / close shelf

Added to `useNavigation` alongside existing arrow key handlers.

- Guard: `phase === 'playing'` (consistent with all other shortcuts)
- `isTouchDevice` check (`navigator.maxTouchPoints > 0`) — no-op on touch devices
- Toggles: opens shelf if closed, closes if open

### Number keys 1–0 — play by position

When the shelf is open and `phase === 'playing'`, keys `1`–`9` and `0` trigger `onPlayByIndex(i)` where `i` is `0`–`9` (`0` key = index 9).

- `onPlayByIndex` is a new callback added to `useNavigation`'s parameter list
- Uses a stable ref pattern (consistent with existing `onNextRef`, etc.)
- Operates on the sorted station array, so the number shown on screen matches the key that plays it
- No-op if `i >= sortedSavedStations.length`
- Also no-op on touch devices

**Number badges in the shelf:**

- Each row from index 0–9 shows a small dim badge: `1`–`9` for indices 0–8, `0` for index 9
- Style: `rgba(255,255,255,0.20)`, `font-size: 10px`, right-aligned, before the remove button
- Hidden on touch devices via `navigator.maxTouchPoints > 0` check (inline style, not CSS media query, to match the existing pattern in this codebase)

---

## 5. Component & Hook Changes

| File | Change |
|---|---|
| `SavedShelf.tsx` | Add sort bar, animated equalizer, drag handles + DnD logic, number badges |
| `useNavigation.ts` | Add `F` shortcut, `1–0` shortcuts, `onPlayByIndex` param |
| `App.tsx` | Extract `useShelfSort` logic; pass `sortedSavedStations` + `onPlayByIndex` to `useNavigation`; pass sort props to `SavedShelf` |
| `useShelfSort.ts` | New hook: manages `sortMode` state (localStorage), `shelfOrder` (localStorage), `sortedStations(stations)` derivation, `appendToOrder`, `setOrder` |

---

## 6. Testing

- `useShelfSort`: sort modes produce correct ordering; drag reorder updates order array; new saves are appended; missing UUIDs fall back to end
- `useNavigation`: F key toggles shelf; number keys call `onPlayByIndex` with correct index; both are no-ops on touch devices
- `SavedShelf`: equalizer renders on current station row; sort bar chips render and respond to click; number badges render for indices 0–9 only
