# Saved Stations Limit — Design Spec

**Date:** 2026-04-26
**Feature:** v1.1 — Saved stations limit (free: 10, Plus: unlimited)

---

## Scope & Behaviour

- Free users can save up to 10 stations. Plus users have no limit.
- Grandfathered users (installed before v1.1) are treated identically to Plus users for save limits. Completely invisible in UI.
- No counter shown anywhere. The limit is silent until hit.
- On the 11th save attempt by a non-eligible user: light haptic tap + "Save limit reached" toast. Toast is tappable — opens Settings to the Plus section. No save is added.

---

## Settings — Plus Upgrade Card

The feature split table gains one new row, positioned to reflect that saving is a core interaction:

| Feature | Free | Plus |
|---|---|---|
| Full catalog + recommendation engine | ✅ | ✅ |
| All gestures, saving, favorites | ✅ (up to 10) | ✅ Unlimited |
| Background playback | ✅ | ✅ |
| All haptics | ✅ | ✅ |
| Default visualization (Drift theme) | ✅ | ✅ |
| Visualization themes | — | ✅ |
| Home screen widget | — | ✅ |
| Sleep timer | — | ✅ |
| Listening stats | — | ✅ |
| Station filter | — | ✅ |
| Station search | — | ✅ |

No changes to the pitch copy above the button — the table row is sufficient.

---

## Data & Logic

**Save eligibility:**
- `FavoritesStore` is the source of truth for the save count — no new storage needed.
- A computed `canSave: Bool` on `AppState` returns `true` if the user is Plus, grandfathered, or `savedStations.count < 10`.
- The swipe-down save path checks `canSave` before adding. If `false`, fires a light haptic tap + "Save limit reached" toast instead. No save is added.
- The tappable toast opens Settings to the Plus section via the existing `AppState` sheet presentation mechanism.

**Grandfathering:**
- Detected via a `hasGrandfatheredUnlimitedSaves` Bool in UserDefaults.
- The check runs as the very first thing in `AppState.init`, before `FavoritesStore` is initialised.
- Detection logic: if the `FavoritesStore` persisted JSON file exists on disk AND `hasGrandfatheredUnlimitedSaves` is absent → user upgraded from v1.0 → write `true`.
- If no prior file exists → fresh install → write `false`.
- Once written, the flag is never changed again.
- The flag has no UI representation — grandfathered users are indistinguishable from Plus users in the save flow.

---

## Testing

- Unit test: saving exactly 10 stations succeeds; the 11th is blocked for a free non-grandfathered user.
- Unit test: grandfathering flag written correctly — simulated upgrade path (favorites file exists before init) sets flag to `true`; fresh install (no file) sets flag to `false`.
- Unit test: Plus user bypasses the limit.
- Unit test: grandfathered user bypasses the limit.
- Toast and haptic are UI-level — no unit tests needed.
