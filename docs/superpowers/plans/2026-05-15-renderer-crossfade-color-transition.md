# Renderer Cross-fade & Color Transition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a 300ms opacity cross-fade between ambient and audio-reactive renderers, plus a 2s HSL color transition between stations, with all visual state owned by VisualizationManager.

**Architecture:** VisualizationManager becomes the single source of truth for color and renderer state. It lerps color in HSL space (shortest hue path, smoothstep easing) and drives both renderers' opacity during cross-fade. Renderers are passive receivers — they expose `setColor(r,g,b)` and `setOpacity(n)` and just draw. Color transition and cross-fade are independent; both run simultaneously when a station changes.

**Tech Stack:** Three.js (Color HSL methods), Web Audio API, Vitest

**Spec:** `docs/superpowers/specs/2026-05-15-renderer-crossfade-color-transition-design.md`

---

## File map

| File | Action | Responsibility |
|---|---|---|
| `src/visualization/AudioReactiveRenderer.ts` | Modify | Rename `setGenreColor` → `setColor(r,g,b)` |
| `src/visualization/AmbientRenderer.ts` | Modify | Remove color transition logic; add `setColor(r,g,b)`; remove `setMetadata` |
| `src/visualization/colorUtils.ts` | Create | Pure HSL lerp + smoothstep helpers |
| `src/visualization/colorUtils.test.ts` | Create | Unit tests for colorUtils |
| `src/visualization/VisualizationManager.ts` | Modify | Color lerp state + HSL push in tick; cross-fade state + dual-render in tick |
| `src/visualization/VisualizationManager.test.ts` | Modify | Update mocks; add color push + cross-fade tests |

---

## Task 1: Rename `setGenreColor` → `setColor` in AudioReactiveRenderer

**Files:**
- Modify: `src/visualization/AudioReactiveRenderer.ts`
- Modify: `src/visualization/VisualizationManager.ts`
- Modify: `src/visualization/VisualizationManager.test.ts`

- [ ] **Step 1: Update the AudioReactiveRenderer mock to use `setColor`**

In `src/visualization/VisualizationManager.test.ts`, change the `AudioReactiveRenderer` mock:

```typescript
vi.mock('./AudioReactiveRenderer', () => ({
  AudioReactiveRenderer: vi.fn(function () {
    return {
      setColor: vi.fn(),
      setFrequencyBuckets: vi.fn(),
      setOpacity: vi.fn(),
      animate: vi.fn(),
      dispose: vi.fn(),
    }
  }),
}))
```

- [ ] **Step 2: Run the existing tests to confirm they fail**

```bash
cd /Users/vitorfreitas/dev/drifting-web && npm run test:run -- --reporter=verbose 2>&1 | tail -30
```

Expected: failures referencing `setGenreColor` not being a function.

- [ ] **Step 3: Rename the method in AudioReactiveRenderer.ts**

Replace the `setGenreColor` method (lines 99–101 of `src/visualization/AudioReactiveRenderer.ts`):

```typescript
setColor(r: number, g: number, b: number): void {
  this.material.uniforms.u_color?.value?.set(r, g, b)
}
```

- [ ] **Step 4: Update the call site in VisualizationManager.ts**

In `src/visualization/VisualizationManager.ts`, inside `loadStation()`, replace:
```typescript
this.audioReactiveRenderer.setGenreColor(getGenreColor(station.tags))
```
with:
```typescript
this.audioReactiveRenderer.setColor(c.r, c.g, c.b)
```

(`c` is already declared at the top of `loadStation` as `const c = getGenreColor(station.tags)`.)

- [ ] **Step 5: Run tests and confirm they pass**

```bash
npm run test:run 2>&1 | tail -10
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/visualization/AudioReactiveRenderer.ts src/visualization/VisualizationManager.ts src/visualization/VisualizationManager.test.ts
git commit -m "refactor: rename AudioReactiveRenderer.setGenreColor → setColor(r,g,b)"
```

---

## Task 2: Simplify AmbientRenderer — remove color transition, add `setColor`

**Files:**
- Modify: `src/visualization/AmbientRenderer.ts`
- Modify: `src/visualization/VisualizationManager.ts`
- Modify: `src/visualization/VisualizationManager.test.ts`

- [ ] **Step 1: Update the AmbientRenderer mock — replace `setMetadata` with `setColor`**

In `src/visualization/VisualizationManager.test.ts`, change:

```typescript
vi.mock('./AmbientRenderer', () => ({
  AmbientRenderer: vi.fn(function () {
    return {
      setColor: vi.fn(),
      setOpacity: vi.fn(),
      animate: vi.fn(),
      dispose: vi.fn(),
    }
  }),
}))
```

- [ ] **Step 2: Run tests to confirm failure**

```bash
npm run test:run 2>&1 | tail -20
```

Expected: failures — `setMetadata` is not a function.

- [ ] **Step 3: Rewrite AmbientRenderer.ts without color transition state**

Replace the entire `AmbientRenderer` class (keep the GLSL shaders and the `callCtor` helper at the top of the file unchanged — only replace the class body):

```typescript
export class AmbientRenderer {
  private scene: THREE.Scene
  private camera: THREE.OrthographicCamera
  private material: THREE.ShaderMaterial

  constructor() {
    this.scene = callCtor(THREE.Scene)
    this.camera = callCtor(THREE.OrthographicCamera, -1, 1, 1, -1, 0, 1)

    this.material = callCtor(THREE.ShaderMaterial, {
      vertexShader: VERT,
      fragmentShader: FRAG,
      transparent: true,
      uniforms: {
        u_time:    { value: 0 },
        u_color:   { value: callCtor(THREE.Vector3, 0.1, 0.6, 0.6) },
        u_opacity: { value: 1 },
      },
    })

    if (typeof this.scene.add === 'function') {
      this.scene.add(callCtor(THREE.Mesh, callCtor(THREE.PlaneGeometry, 2, 2), this.material))
    }
  }

  setColor(r: number, g: number, b: number): void {
    this.material.uniforms.u_color.value.set(r, g, b)
  }

  setOpacity(opacity: number): void {
    this.material.uniforms.u_opacity.value = opacity
  }

  animate(renderer: THREE.WebGLRenderer, elapsedMs: number): void {
    this.material.uniforms.u_time.value = elapsedMs / 1000
    renderer.render(this.scene, this.camera)
  }

  dispose(): void {
    this.material.dispose()
  }
}
```

- [ ] **Step 4: Update `loadStation` in VisualizationManager.ts — replace `setMetadata` with direct `setColor`**

The full updated `loadStation` method (this is an interim state; Tasks 4–5 will replace it again):

```typescript
loadStation(station: Station, player: AudioPlayer): void {
  const c = getGenreColor(station.tags)
  this.currentColor.setRGB(c.r, c.g, c.b)
  this.currentPlayer = player
  if (this.silenceTimer) {
    clearTimeout(this.silenceTimer)
    this.silenceTimer = null
  }

  if (station.corsFriendly) {
    this.activeMode = 'audio-reactive'
    this.audioReactiveRenderer.setColor(c.r, c.g, c.b)
    this.silenceTimer = setTimeout(() => {
      if (this.currentPlayer?.isSilent()) {
        this.activeMode = 'ambient'
        this.ambientRenderer.setColor(c.r, c.g, c.b)
      }
    }, SILENCE_GRACE_MS)
  } else {
    this.activeMode = 'ambient'
    this.ambientRenderer.setColor(c.r, c.g, c.b)
  }
}
```

- [ ] **Step 5: Run tests and confirm all pass**

```bash
npm run test:run 2>&1 | tail -10
```

- [ ] **Step 6: Commit**

```bash
git add src/visualization/AmbientRenderer.ts src/visualization/VisualizationManager.ts src/visualization/VisualizationManager.test.ts
git commit -m "refactor: simplify AmbientRenderer — remove color lerp, add setColor(r,g,b)"
```

---

## Task 3: Add HSL color interpolation utilities

**Files:**
- Create: `src/visualization/colorUtils.ts`
- Create: `src/visualization/colorUtils.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `src/visualization/colorUtils.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import * as THREE from 'three'
import { smoothstep, lerpColorHSL } from './colorUtils'

describe('smoothstep', () => {
  it('returns 0 at t=0', () => expect(smoothstep(0)).toBe(0))
  it('returns 1 at t=1', () => expect(smoothstep(1)).toBe(1))
  it('returns 0.5 at t=0.5', () => expect(smoothstep(0.5)).toBe(0.5))
  it('clamps values below 0', () => expect(smoothstep(-1)).toBe(0))
  it('clamps values above 1', () => expect(smoothstep(2)).toBe(1))
  it('is slower than linear at the midpoint (S-curve shape)', () => {
    expect(smoothstep(0.25)).toBeLessThan(0.25)
    expect(smoothstep(0.75)).toBeGreaterThan(0.75)
  })
})

describe('lerpColorHSL', () => {
  it('returns the from color when t=0', () => {
    const from = new THREE.Color().setHSL(0.17, 1, 0.5)
    const to   = new THREE.Color().setHSL(0.67, 1, 0.5)
    const out  = new THREE.Color()
    lerpColorHSL(from, to, 0, out)
    const hsl = { h: 0, s: 0, l: 0 }
    out.getHSL(hsl)
    expect(hsl.h).toBeCloseTo(0.17, 2)
    expect(hsl.s).toBeCloseTo(1, 2)
    expect(hsl.l).toBeCloseTo(0.5, 2)
  })

  it('returns the to color when t=1', () => {
    const from = new THREE.Color().setHSL(0.17, 1, 0.5)
    const to   = new THREE.Color().setHSL(0.67, 1, 0.5)
    const out  = new THREE.Color()
    lerpColorHSL(from, to, 1, out)
    const hsl = { h: 0, s: 0, l: 0 }
    out.getHSL(hsl)
    expect(hsl.h).toBeCloseTo(0.67, 2)
  })

  it('takes the shortest hue path — yellow to blue passes through green, not red', () => {
    // Yellow hue ~0.17, blue hue ~0.67, midpoint via green ~0.42
    // Wrong direction (through red) would give ~0.92
    const from = new THREE.Color().setHSL(0.17, 1, 0.5)
    const to   = new THREE.Color().setHSL(0.67, 1, 0.5)
    const out  = new THREE.Color()
    lerpColorHSL(from, to, 0.5, out)
    const hsl = { h: 0, s: 0, l: 0 }
    out.getHSL(hsl)
    expect(hsl.h).toBeCloseTo(0.42, 1)
  })

  it('wraps correctly when shortest path crosses the 0/1 hue boundary', () => {
    // Red hue 0.02, magenta hue 0.92 — shortest path goes backward through 0
    // Midpoint should be ~0.97, not ~0.47 (long way around)
    const from = new THREE.Color().setHSL(0.02, 1, 0.5)
    const to   = new THREE.Color().setHSL(0.92, 1, 0.5)
    const out  = new THREE.Color()
    lerpColorHSL(from, to, 0.5, out)
    const hsl = { h: 0, s: 0, l: 0 }
    out.getHSL(hsl)
    expect(hsl.h).toBeCloseTo(0.97, 1)
  })

  it('interpolates saturation and lightness linearly', () => {
    const from = new THREE.Color().setHSL(0.0, 0.2, 0.3)
    const to   = new THREE.Color().setHSL(0.0, 0.8, 0.7)
    const out  = new THREE.Color()
    lerpColorHSL(from, to, 0.5, out)
    const hsl = { h: 0, s: 0, l: 0 }
    out.getHSL(hsl)
    expect(hsl.s).toBeCloseTo(0.5, 2)
    expect(hsl.l).toBeCloseTo(0.5, 2)
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npm run test:run -- src/visualization/colorUtils.test.ts 2>&1 | tail -15
```

Expected: `Cannot find module './colorUtils'`.

- [ ] **Step 3: Implement colorUtils.ts**

Create `src/visualization/colorUtils.ts`:

```typescript
interface HSLTarget {
  h: number
  s: number
  l: number
}

interface HSLColor {
  getHSL(target: HSLTarget): HSLTarget
  setHSL(h: number, s: number, l: number): void
}

export function smoothstep(t: number): number {
  const c = Math.max(0, Math.min(1, t))
  return c * c * (3 - 2 * c)
}

export function lerpColorHSL(from: HSLColor, to: HSLColor, t: number, out: HSLColor): void {
  const f = { h: 0, s: 0, l: 0 }
  const tgt = { h: 0, s: 0, l: 0 }
  from.getHSL(f)
  to.getHSL(tgt)

  // Shortest-path hue: normalize difference to [0,1) then fold to (-0.5, 0.5]
  let delta = tgt.h - f.h
  delta = ((delta % 1) + 1) % 1
  if (delta > 0.5) delta -= 1

  const h = ((f.h + delta * t) + 1) % 1
  const s = f.s + (tgt.s - f.s) * t
  const l = f.l + (tgt.l - f.l) * t
  out.setHSL(h, s, l)
}
```

- [ ] **Step 4: Run tests and confirm all pass**

```bash
npm run test:run -- src/visualization/colorUtils.test.ts 2>&1 | tail -10
```

Expected: all 9 tests pass.

- [ ] **Step 5: Run full suite to confirm no regressions**

```bash
npm run test:run 2>&1 | tail -10
```

- [ ] **Step 6: Commit**

```bash
git add src/visualization/colorUtils.ts src/visualization/colorUtils.test.ts
git commit -m "feat: add HSL color interpolation utility (smoothstep + lerpColorHSL)"
```

---

## Task 4: Wire color transition into VisualizationManager

**Files:**
- Modify: `src/visualization/VisualizationManager.ts`
- Modify: `src/visualization/VisualizationManager.test.ts`

- [ ] **Step 1: Update the THREE.Color mock to support HSL methods**

In `src/visualization/VisualizationManager.test.ts`, update the `Color` entry in the `three` mock (keep everything else the same):

```typescript
Color: vi.fn(() => ({
  r: 0.1, g: 0.6, b: 0.6,
  setRGB: vi.fn(),
  getHSL: vi.fn((out: { h: number; s: number; l: number }) => {
    out.h = 0; out.s = 0; out.l = 0.5
    return out
  }),
  setHSL: vi.fn(),
  copy: vi.fn().mockReturnThis(),
})),
```

Also add imports for the mocked renderers at the top of the test file (after the existing `import { VisualizationManager }`):

```typescript
import { AmbientRenderer } from './AmbientRenderer'
import { AudioReactiveRenderer } from './AudioReactiveRenderer'
```

- [ ] **Step 2: Write the failing test for color being pushed to the renderer each tick**

Add this describe block to `src/visualization/VisualizationManager.test.ts`:

```typescript
describe('VisualizationManager color transition', () => {
  it('pushes current color to the active renderer on each tick', () => {
    let rafCb: FrameRequestCallback = () => {}
    vi.stubGlobal('requestAnimationFrame', vi.fn((cb: FrameRequestCallback) => {
      rafCb = cb
      return 1
    }))

    const vm = new VisualizationManager(makeCanvas())
    vm.start()
    vm.loadStation(makeStation(false), makePlayer(false))

    const ambientMock = vi.mocked(AmbientRenderer).mock.results.at(-1)!.value
    vi.clearAllMocks()

    rafCb(0)

    expect(ambientMock.setColor).toHaveBeenCalled()
  })
})
```

- [ ] **Step 3: Run to confirm the test fails**

```bash
npm run test:run -- src/visualization/VisualizationManager.test.ts 2>&1 | tail -20
```

Expected: `ambientMock.setColor` not called (it's called in loadStation directly, not in tick).

- [ ] **Step 4: Add color transition state and imports to VisualizationManager.ts**

At the top of `src/visualization/VisualizationManager.ts`, add the import:

```typescript
import { smoothstep, lerpColorHSL } from './colorUtils'
```

Inside the `VisualizationManager` class, add these fields after `private currentColor`:

```typescript
private colorFrom: THREE.Color
private colorTarget: THREE.Color
private colorTransitionStart = -Infinity
private readonly COLOR_TRANSITION_MS = 2000
```

In the constructor, initialize them after `this.currentColor`:

```typescript
this.colorFrom = callCtor(THREE.Color, 0.1, 0.6, 0.6)
this.colorTarget = callCtor(THREE.Color, 0.1, 0.6, 0.6)
```

- [ ] **Step 5: Replace `loadStation` to use color transition state**

Replace the entire `loadStation` method in `src/visualization/VisualizationManager.ts`:

```typescript
loadStation(station: Station, player: AudioPlayer): void {
  this.colorFrom.copy(this.currentColor)
  const c = getGenreColor(station.tags)
  this.colorTarget.setRGB(c.r, c.g, c.b)
  this.colorTransitionStart = performance.now()

  this.currentPlayer = player
  if (this.silenceTimer) {
    clearTimeout(this.silenceTimer)
    this.silenceTimer = null
  }

  if (station.corsFriendly) {
    this.activeMode = 'audio-reactive'
    this.silenceTimer = setTimeout(() => {
      if (this.currentPlayer?.isSilent()) {
        this.activeMode = 'ambient'
      }
    }, SILENCE_GRACE_MS)
  } else {
    this.activeMode = 'ambient'
  }
}
```

- [ ] **Step 6: Update `tick` to advance color and push to the active renderer**

Replace the `tick` method in `src/visualization/VisualizationManager.ts`:

```typescript
private tick(): void {
  const now = performance.now()
  const deltaMs = this.clock.getDelta() * 1000
  this.elapsedMs += deltaMs

  const ct = smoothstep(Math.min((now - this.colorTransitionStart) / this.COLOR_TRANSITION_MS, 1))
  lerpColorHSL(this.colorFrom, this.colorTarget, ct, this.currentColor)
  const { r, g, b } = this.currentColor

  this.webglRenderer.autoClear = true

  if (this.activeMode === 'audio-reactive') {
    if (this.currentPlayer) {
      this.audioReactiveRenderer.setFrequencyBuckets(this.currentPlayer.getFrequencyBuckets())
    }
    this.audioReactiveRenderer.setColor(r, g, b)
    this.audioReactiveRenderer.setOpacity(1)
    this.audioReactiveRenderer.animate(this.webglRenderer, deltaMs)
  } else {
    this.ambientRenderer.setColor(r, g, b)
    this.ambientRenderer.setOpacity(1)
    this.ambientRenderer.animate(this.webglRenderer, this.elapsedMs)
  }

  this.burst.setColor(this.currentColor)
  this.burst.update(deltaMs / 1000)
  this.burst.render(this.webglRenderer)
}
```

- [ ] **Step 7: Run tests and confirm all pass**

```bash
npm run test:run 2>&1 | tail -10
```

- [ ] **Step 8: Commit**

```bash
git add src/visualization/VisualizationManager.ts src/visualization/VisualizationManager.test.ts
git commit -m "feat: wire HSL color transition into VisualizationManager (2s, tick-driven)"
```

---

## Task 5: Wire renderer cross-fade into VisualizationManager

**Files:**
- Modify: `src/visualization/VisualizationManager.ts`
- Modify: `src/visualization/VisualizationManager.test.ts`

- [ ] **Step 1: Write the failing cross-fade tests**

Add this describe block to `src/visualization/VisualizationManager.test.ts`:

```typescript
describe('VisualizationManager cross-fade', () => {
  it('renders both renderers with complementary opacities during a mode switch', () => {
    let rafCb: FrameRequestCallback = () => {}
    vi.stubGlobal('requestAnimationFrame', vi.fn((cb: FrameRequestCallback) => {
      rafCb = cb
      return 1
    }))

    const vm = new VisualizationManager(makeCanvas())
    vm.start()

    const ambientMock = vi.mocked(AmbientRenderer).mock.results.at(-1)!.value
    const audioMock = vi.mocked(AudioReactiveRenderer).mock.results.at(-1)!.value

    // Freeze performance.now at 1000 so crossfadeStart = 1000
    const nowSpy = vi.spyOn(performance, 'now').mockReturnValue(1000)
    // Load CORS station → switches from ambient to audio-reactive, triggering cross-fade
    vm.loadStation(makeStation(true), makePlayer(false))

    // Advance to 150ms into the 300ms fade → progress = 0.5
    nowSpy.mockReturnValue(1150)
    vi.clearAllMocks()
    rafCb(0)

    // Both renderers should receive opacity calls that sum to 1
    expect(ambientMock.setOpacity).toHaveBeenLastCalledWith(expect.closeTo(0.5, 1))
    expect(audioMock.setOpacity).toHaveBeenLastCalledWith(expect.closeTo(0.5, 1))
  })

  it('stops rendering the outgoing renderer after the cross-fade completes', () => {
    let rafCb: FrameRequestCallback = () => {}
    vi.stubGlobal('requestAnimationFrame', vi.fn((cb: FrameRequestCallback) => {
      rafCb = cb
      return 1
    }))

    const vm = new VisualizationManager(makeCanvas())
    vm.start()

    const ambientMock = vi.mocked(AmbientRenderer).mock.results.at(-1)!.value

    const nowSpy = vi.spyOn(performance, 'now').mockReturnValue(1000)
    vm.loadStation(makeStation(true), makePlayer(false))

    // Jump past the 300ms fade
    nowSpy.mockReturnValue(1400)
    vi.clearAllMocks()
    rafCb(0)

    // Ambient (outgoing) should not be animated after fade completes
    expect(ambientMock.animate).not.toHaveBeenCalled()
  })

  it('does not trigger a cross-fade when mode stays the same (non-CORS → non-CORS)', () => {
    let rafCb: FrameRequestCallback = () => {}
    vi.stubGlobal('requestAnimationFrame', vi.fn((cb: FrameRequestCallback) => {
      rafCb = cb
      return 1
    }))

    const vm = new VisualizationManager(makeCanvas())
    vm.start()

    const audioMock = vi.mocked(AudioReactiveRenderer).mock.results.at(-1)!.value

    vi.spyOn(performance, 'now').mockReturnValue(1000)
    vm.loadStation(makeStation(false), makePlayer(false)) // non-CORS → ambient
    vm.loadStation(makeStation(false), makePlayer(false)) // non-CORS → ambient (no switch)

    vi.clearAllMocks()
    rafCb(0)

    // Audio-reactive renderer should not be called at all
    expect(audioMock.animate).not.toHaveBeenCalled()
  })
})
```

- [ ] **Step 2: Run to confirm the tests fail**

```bash
npm run test:run -- src/visualization/VisualizationManager.test.ts 2>&1 | tail -20
```

Expected: cross-fade tests fail (no dual-render logic yet).

- [ ] **Step 3: Add cross-fade state fields to VisualizationManager.ts**

Inside the `VisualizationManager` class, add after the color transition fields:

```typescript
private crossfadeFrom: RendererMode | null = null
private crossfadeStart = -Infinity
private readonly CROSSFADE_MS = 300
```

- [ ] **Step 4: Update `loadStation` to trigger cross-fade on mode switch**

Replace `loadStation` with the final version:

```typescript
loadStation(station: Station, player: AudioPlayer): void {
  this.colorFrom.copy(this.currentColor)
  const c = getGenreColor(station.tags)
  this.colorTarget.setRGB(c.r, c.g, c.b)
  this.colorTransitionStart = performance.now()

  this.currentPlayer = player
  if (this.silenceTimer) {
    clearTimeout(this.silenceTimer)
    this.silenceTimer = null
  }

  const newMode: RendererMode = station.corsFriendly ? 'audio-reactive' : 'ambient'
  if (newMode !== this.activeMode) {
    this.crossfadeFrom = this.activeMode
    this.crossfadeStart = performance.now()
    this.activeMode = newMode
  }

  if (station.corsFriendly) {
    this.silenceTimer = setTimeout(() => {
      if (this.currentPlayer?.isSilent()) {
        if (this.activeMode !== 'ambient') {
          this.crossfadeFrom = this.activeMode
          this.crossfadeStart = performance.now()
          this.activeMode = 'ambient'
        }
      }
    }, SILENCE_GRACE_MS)
  }
}
```

- [ ] **Step 5: Update `tick` to handle cross-fade dual-render**

Replace the `tick` method:

```typescript
private tick(): void {
  const now = performance.now()
  const deltaMs = this.clock.getDelta() * 1000
  this.elapsedMs += deltaMs

  const ct = smoothstep(Math.min((now - this.colorTransitionStart) / this.COLOR_TRANSITION_MS, 1))
  lerpColorHSL(this.colorFrom, this.colorTarget, ct, this.currentColor)
  const { r, g, b } = this.currentColor

  if (this.currentPlayer) {
    this.audioReactiveRenderer.setFrequencyBuckets(this.currentPlayer.getFrequencyBuckets())
  }

  if (this.crossfadeFrom !== null) {
    const progress = Math.min((now - this.crossfadeStart) / this.CROSSFADE_MS, 1)

    // Outgoing renderer fades out
    const outgoing = this.crossfadeFrom === 'ambient' ? this.ambientRenderer : this.audioReactiveRenderer
    outgoing.setColor(r, g, b)
    outgoing.setOpacity(1 - progress)
    this.webglRenderer.autoClear = true
    if (this.crossfadeFrom === 'ambient') {
      this.ambientRenderer.animate(this.webglRenderer, this.elapsedMs)
    } else {
      this.audioReactiveRenderer.animate(this.webglRenderer, deltaMs)
    }

    // Incoming renderer fades in
    const incoming = this.activeMode === 'ambient' ? this.ambientRenderer : this.audioReactiveRenderer
    incoming.setColor(r, g, b)
    incoming.setOpacity(progress)
    this.webglRenderer.autoClear = false
    if (this.activeMode === 'ambient') {
      this.ambientRenderer.animate(this.webglRenderer, this.elapsedMs)
    } else {
      this.audioReactiveRenderer.animate(this.webglRenderer, deltaMs)
    }

    if (progress >= 1) this.crossfadeFrom = null
  } else {
    this.webglRenderer.autoClear = true
    if (this.activeMode === 'audio-reactive') {
      this.audioReactiveRenderer.setColor(r, g, b)
      this.audioReactiveRenderer.setOpacity(1)
      this.audioReactiveRenderer.animate(this.webglRenderer, deltaMs)
    } else {
      this.ambientRenderer.setColor(r, g, b)
      this.ambientRenderer.setOpacity(1)
      this.ambientRenderer.animate(this.webglRenderer, this.elapsedMs)
    }
  }

  this.burst.setColor(this.currentColor)
  this.burst.update(deltaMs / 1000)
  this.burst.render(this.webglRenderer)
}
```

- [ ] **Step 6: Run all tests and confirm they pass**

```bash
npm run test:run 2>&1 | tail -15
```

Expected: all tests pass, including the 3 new cross-fade tests.

- [ ] **Step 7: Run lint**

```bash
npm run lint 2>&1 | tail -10
```

Fix any lint errors before committing.

- [ ] **Step 8: Commit**

```bash
git add src/visualization/VisualizationManager.ts src/visualization/VisualizationManager.test.ts
git commit -m "feat: add 300ms renderer cross-fade in VisualizationManager"
```

---

## Self-review notes

- **lerpColorHSL shortest-path formula**: the spec listed `((targetH - fromH + 1.5) % 1) - 0.5` which fails for the yellow→blue case (exactly 0.5 apart). The plan uses the correct formula: normalize delta to [0,1), then fold to (−0.5, 0.5]. Tests cover both the yellow→blue case and the hue wrap-around case.
- **setMetadata removal**: removed from AmbientRenderer and its call site in VisualizationManager (not just stubbed). The manager calls `getGenreColor` directly.
- **autoClear discipline**: BurstEffect already manages its own `autoClear` (sets false, renders, sets true). The cross-fade code sets true/false only for the main renderers; BurstEffect's render always composites on top correctly.
- **No cross-fade on same-mode switch**: non-CORS → non-CORS stays in ambient; only `activeMode` changes trigger `crossfadeFrom`.
