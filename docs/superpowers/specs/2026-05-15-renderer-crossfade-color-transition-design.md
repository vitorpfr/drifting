# Renderer Cross-fade & Color Transition — Design Spec

**Date:** 2026-05-15
**Status:** Approved
**Feature:** Smooth 300ms renderer cross-fade + 2s HSL background color transition between stations

---

## Problem

Station changes currently produce two hard cuts:

1. **Renderer switch** — ambient ↔ audio-reactive flips instantly with no blend.
2. **Color change** — `AmbientRenderer` already lerps color over 2s but uses a buggy cumulative lerp and RGB interpolation. `AudioReactiveRenderer` snaps color immediately. Neither is consistent.

---

## Goals

- 300ms opacity cross-fade whenever the active renderer changes.
- 2s HSL color transition between stations on both renderers, with hue taking the shortest path around the color wheel.
- Architecture that supports future themes (color palettes, form parameters, renderer swaps) without restructuring.

---

## Architecture

### Principle: VisualizationManager owns all visual state

`VisualizationManager` is the single source of truth for color and renderer state. Renderers are dumb — they receive color and opacity each tick and draw. They do not own transition logic.

This scales to future themes: swap `colorTarget: THREE.Color` for `targetTheme: Theme` and lerp all theme properties in `tick()`. For form-based themes, the manager either pushes a `morphT` parameter to the renderer or activates a different renderer class entirely — both patterns already exist.

---

## Color Transition

### Location
`VisualizationManager`

### State added
```ts
private colorFrom: THREE.Color        // snapshot of color at transition start
private colorTarget: THREE.Color      // destination color
private colorTransitionStart: number  // performance.now() at start, -Infinity if idle
private readonly COLOR_TRANSITION_MS = 2000
```

### Trigger
`loadStation()` — on every station change:
1. Snapshot the current interpolated color into `colorFrom`.
2. Call `getGenreColor(station.tags)` and store result in `colorTarget`.
3. Set `colorTransitionStart = performance.now()`.

### Tick behavior
```
t = clamp((now - colorTransitionStart) / COLOR_TRANSITION_MS, 0, 1)
eased = smoothstep(t)           // t * t * (3 - 2t)
currentColor = lerpHSL(colorFrom, colorTarget, eased)
push currentColor to active renderer(s) via setColor()
```

### HSL interpolation
`THREE.Color.getHSL()` / `setHSL()` used per tick. Hue takes the shortest path:
```
delta = ((targetH - fromH + 1.5) % 1) - 0.5
currentH = fromH + delta * eased
```
This ensures yellow → blue sweeps through green rather than wrapping the long way.

### What changes in AmbientRenderer
Remove: `targetColor`, `currentColor`, `colorTransitionStart`, `colorTransitionDuration`, color lerp in `animate()`, `getGenreColor` call in `setMetadata()`.

Add: `setColor(r: number, g: number, b: number)` — sets `u_color` uniform directly.

`setMetadata()` is removed entirely. The manager calls `getGenreColor` directly and no longer delegates to the renderer. The call site in `VisualizationManager.loadStation()` is removed with it.

---

## Renderer Cross-fade

### Location
`VisualizationManager`

### State added
```ts
private crossfadeFrom: RendererMode | null = null
private crossfadeStart = -Infinity
private readonly CROSSFADE_MS = 300
```

### Trigger
Any time `activeMode` changes — in `loadStation()` (CORS station vs non-CORS) or in the silence fallback timer. Before updating `activeMode`:
```
crossfadeFrom = activeMode          // record outgoing renderer
crossfadeStart = performance.now()
activeMode = newMode                // set incoming renderer
```

### Tick behavior
```
if crossfadeFrom is not null:
  progress = clamp((now - crossfadeStart) / CROSSFADE_MS, 0, 1)
  
  render outgoing renderer:
    setOpacity(1 - progress)
    autoClear = true
    call animate()

  render incoming renderer:
    setOpacity(progress)
    autoClear = false               // don't wipe the first render
    call animate()

  if progress >= 1: crossfadeFrom = null

else:
  render activeMode renderer at opacity 1 (autoClear = true)
```

### Interaction with color transition
The two transitions are fully independent. Color lerps over 2s; renderer cross-fades over 300ms. During cross-fade, the manager pushes the current interpolated color to both renderers simultaneously. No coordination needed.

### Cases that do NOT trigger a cross-fade
- Non-CORS station → non-CORS station: both use ambient renderer, no mode switch.
- Only a color lerp fires in this case.

---

## AudioReactiveRenderer changes

Rename `setGenreColor(color: Color)` → `setColor(r: number, g: number, b: number)` to match `AmbientRenderer`'s new interface. Manager calls this each tick with the current interpolated color.

---

## File change summary

| File | Change |
|---|---|
| `VisualizationManager.ts` | Add color lerp state + HSL interpolation; add cross-fade state + dual-render logic in tick |
| `AmbientRenderer.ts` | Remove color transition state/logic; add `setColor(r,g,b)` |
| `AudioReactiveRenderer.ts` | Rename `setGenreColor` → `setColor(r,g,b)` |
| `genreColors.ts` | No change |
| `types.ts` | No change |

---

## Testing

- `VisualizationManager.test.ts`: cross-fade opacity sequence (both renderers called, opacities sum to 1 during fade); color lerp advances correctly; HSL hue takes short path.
- `AmbientRenderer.test.ts`: drop color-transition tests (logic moved); add `setColor` test.
- `AudioReactiveRenderer.test.ts`: update `setGenreColor` → `setColor` references.

---

## Out of scope

- Theme system (future work — this design is the foundation).
- Form-based transitions.
- Easing curve customization per theme.
