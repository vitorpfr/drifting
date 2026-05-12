# Info Overlay Controls — Design Spec

**Date:** 2026-05-12
**Status:** Approved
**iOS reference:** `design-spec.md` §5 (Info overlay)

---

## Overview

Add always-visible controls to the playing state: share and settings in the top-right corner, play/pause in the bottom-right. Adds a settings modal with three actions. All controls are edge-positioned, dim, and non-intrusive — the visualization remains the focus.

---

## Layout

Three positions, all visible only while `phase === 'playing'`:

| Position | Contents |
|---|---|
| Top-right (`top: 32px, right: 32px`) | Share icon · Settings icon, flex row, 20px gap |
| Bottom-right (`bottom: 40px, right: 40px`) | Play/pause icon |

**Button styling:**
- Tap target: 28×28px
- Icon color: `rgba(255,255,255,0.4)` at rest → `rgba(255,255,255,0.75)` on hover
- No background, no border
- Icons: inline SVG (no icon library)
- `pointer-events: auto`

**New component:** `src/OverlayControls.tsx`
- Props: `onShare`, `onOpenSettings`, `onPlayPause`, `paused: boolean`
- Renders top-right and bottom-right groups

**Play/pause reactive state:**
Currently `isPaused()` is read from a ref and is not reactive. Add `const [paused, setPaused] = useState(false)` to `App.tsx`. Update it in `handlePlayPause` after calling `pause()` / `resume()`.

---

## Share

- Copies `window.location.href` to clipboard via `navigator.clipboard.writeText()`
- Shows existing toast: "Link copied"
- No new component — handled inline in `OverlayControls` via `onShare` prop

**Future (out of scope):** Generate station deep links (`/?station=<uuid>`) once URL routing exists. Requires router + deep link handling on load. Note this in the spec doc so the pattern is established.

---

## Settings Modal

**New component:** `src/SettingsModal.tsx`
- Props: `onClose`, `audioOnly: boolean`, `onToggleAudioOnly`, `onResetProfile`, `onClearSaved`
- Destructive actions call their prop callback; `App.tsx` handles IDB + toast + `onClose`

Opens centered over everything (`z-index` above all other overlays). Music keeps playing.

**Backdrop:** `rgba(0,0,0,0.6)`, covers full viewport. Click dismisses.

**Card:** `max-width: 320px`, centered, dark background (`rgba(15,15,15,0.95)`), `border-radius: 16px`, padding `28px 24px`. × button top-right dismisses.

**Typography:** same dim-white palette as the rest of the UI (`rgba(255,255,255,0.85)` for labels, `rgba(255,255,255,0.4)` for descriptions).

### Three rows

**1. Reset taste profile**
- Label: "Taste profile"
- Description: "Clears your listening history and restarts recommendations from scratch."
- Action: "Reset" button (right-aligned, destructive red tint `rgba(255,80,80,0.8)`)
- On confirm: calls `clearProfile()` from `useRecommendation`, shows toast "Taste profile reset", closes modal

**2. Audio only**
- Label: "Audio only"
- Description: "Pauses the visualization to save battery."
- Control: toggle switch (right-aligned)
- State lives in `App.tsx` as `audioOnly: boolean`, passed to `SettingsModal` and `VisualizationManager`
- On toggle on: calls `vizRef.current?.setAudioOnly(true)` — stops animation loop, renders static radial gradient in station's current color
- On toggle off: calls `vizRef.current?.setAudioOnly(false)` — resumes animation loop
- `VisualizationManager.setAudioOnly(enabled: boolean)` is a new public method

**3. Clear saved stations**
- Label: "Saved stations"
- Description: "Removes all saved stations."
- Action: "Clear all" button (right-aligned, destructive red tint)
- On confirm: calls `clearAll()` from `useSavedStations`, shows toast "Saved stations cleared", closes modal

### Confirmation pattern
Destructive actions ("Reset" and "Clear all") do not require a second confirmation dialog — the button text is explicit enough. The toast serves as the confirmation signal.

---

## New methods required

| Location | Method | What it does |
|---|---|---|
| `useRecommendation` | `clearProfile()` | Deletes the IDB recommendations store entry; resets in-memory state |
| `useSavedStations` | `clearAll()` | Deletes all saved stations from IDB; resets in-memory list |
| `VisualizationManager` | `setAudioOnly(enabled: boolean)` | Stops/resumes animation loop; shows static gradient when enabled |

---

## Files changed

| File | Action |
|---|---|
| `src/OverlayControls.tsx` | New |
| `src/SettingsModal.tsx` | New |
| `src/App.tsx` | Add `paused` state, `audioOnly` state, wire `OverlayControls` and `SettingsModal` |
| `src/useRecommendation.ts` | Add `clearProfile()` |
| `src/useRecommendation.test.ts` | Add tests for `clearProfile()` |
| `src/useSavedStations.ts` | Add `clearAll()` |
| `src/useSavedStations.test.ts` | Add tests for `clearAll()` |
| `src/visualization/VisualizationManager.ts` | Add `setAudioOnly()` |
| `src/visualization/VisualizationManager.test.ts` | Add tests for `setAudioOnly()` |

---

## Out of scope

- Deep link generation and routing (noted for future)
- AirPlay (not applicable to web)
- Search / filter (Plus feature, deferred)
- Haptics settings (not applicable to web)
- Listening stats
- Themes
