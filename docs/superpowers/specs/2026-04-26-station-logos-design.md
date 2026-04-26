# Station Logos / Artwork — Design Spec

**Date:** 2026-04-26
**Feature:** v1.1 — Show station favicon in InfoOverlay

---

## Scope & Behaviour

Show a small station logo in the InfoOverlay to the left of the station name. The logo is sourced from the Radio Browser API's `favicon` field. When no favicon is available (nil URL, load failure, or empty string), a radial gradient tile constructed from the station's `ColorIdentity` is shown instead — the same visual language used in the lock screen and Dynamic Island artwork.

The gradient fallback is rendered immediately on station change. If a favicon URL exists, `AsyncImage` loads it asynchronously and fades it in on success. The user always sees something intentional — there is never a blank or spinner.

---

## Data Layer

**`Station` model (`drifting/drifting/Models/Station.swift`):**

- Add `faviconURL: URL?` — optional field
- CodingKey: `"favicon"`
- Decoding: decode as optional `String`; convert to `URL` only if non-empty, parseable, and scheme is `http` or `https` — otherwise `nil`. This prevents `file://`, `javascript:`, and other non-web schemes from being passed to `AsyncImage`.
- Encoding: use `encodeIfPresent` — omit the key entirely when nil; decoder handles missing key → nil, so cached JSON without the field remains valid

---

## `StationLogoView`

New file: `drifting/drifting/UI/StationLogoView.swift`

- Takes `station: Station` and `size: CGFloat` (default `32`)
- Always renders a radial gradient background immediately using `ColorIdentity(stationID: station.id)` — replicates the sphere illusion from `NowPlayingManager.makeArtwork`: secondary color at a slightly offset centre, fading to primary
- When `station.faviconURL` is non-nil, layers an `AsyncImage` on top:
  - `.empty` / `.failure` → gradient shows through (no image rendered)
  - `.success(image)` → image rendered with `.aspectRatio(contentMode: .fill)`, fades in via `.transition(.opacity)` inside `withAnimation`
- Entire view clipped to `RoundedRectangle(cornerRadius: 6)`

---

## InfoOverlay Integration

**`InfoOverlay` (`drifting/drifting/UI/InfoOverlay.swift`):**

The bottom-left station info block becomes an `HStack(alignment: .top, spacing: 8)`:
- Left: `StationLogoView(station: station, size: 32)`
- Right: existing `VStack` (track title, station name row, location/genre row) — unchanged

The logo aligns to `.top` of the HStack so it sits level with the station name line, not vertically centred across all three lines.

No other layout changes.

---

## Testing

**`StationModelTests.swift` additions:**

- `test_station_validFaviconURL_decoded` — a JSON payload with a valid `favicon` string decodes to a non-nil `faviconURL`
- `test_station_emptyFaviconString_decodesToNil` — an empty `"favicon": ""` decodes to `nil`
- `test_station_missingFaviconKey_decodesToNil` — a JSON payload without the `favicon` key decodes to `nil`
- `test_station_nonHttpFaviconURL_decodesToNil` — a `favicon` with a `file://` or `javascript:` scheme decodes to `nil`

No tests for `StationLogoView` — pure SwiftUI view with no logic; `AsyncImage` loading is not unit-testable.
