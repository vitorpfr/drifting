# Drifting Web — Visualization Design

**Created:** 2026-05-09
**Status:** Approved

---

## Context

The Drifting web port needs a visualization strategy. The iOS app uses a Metal-based audio-reactive visualization (bass/mid/treble FFT-driven) at 120fps. On web, audio analysis requires CORS headers from the station stream — without them, `AnalyserNode` returns silence. Approximately 73% of the filtered catalog (≥128 kbps, music genre, active stations) returns valid CORS headers.

The chosen approach is a hybrid dual-mode visualization: audio-reactive when CORS is available, ambient generative when it is not. The two modes have visually distinct personalities so the user always knows which mode they're in without any explicit label.

---

## Tech Stack

- **Framework:** React + Vite
- **Rendering:** Three.js (WebGL) for both renderers
- **Audio analysis:** Web Audio API `AnalyserNode` (audio-reactive mode only)

Three.js runs a full-screen WebGL canvas managed entirely outside React. React renders the UI overlay (station info, controls) as a separate DOM layer on top. This keeps the visualization code framework-agnostic.

---

## Architecture

Three components:

**`AudioReactiveRenderer`**
Uses `AnalyserNode` FFT data to drive a GPU-computed particle system via custom GLSL vertex/fragment shaders. Receives bass/mid/treble frequency buckets each frame. Sharp, geometric, energetic visual personality.

**`AmbientRenderer`**
Pure generative animation. No audio input. Perlin/Simplex noise flow field computed on the GPU in GLSL. Dense slow-moving particle trails. Receives station metadata (genre, country) at load time via shader uniforms. Warm, fluid, organic visual personality.

**`VisualizationManager`**
Owns the Three.js canvas. Listens for station changes. Selects which renderer is active based on the station's `corsFriendly` flag. Handles transitions between renderers. Detects silent `AnalyserNode` as a runtime CORS fallback.

---

## Mode Selection

At station load time:

1. Check station's `corsFriendly` flag (precomputed — see catalog job below).
2. If `true`: start `AudioReactiveRenderer`, connect `AnalyserNode` to the audio stream.
3. If `false`: start `AmbientRenderer` with genre/country metadata.

**Runtime fallback:** After switching to `AudioReactiveRenderer`, sample `AnalyserNode` for 2 seconds. If all frequency bins remain zero (CORS failed silently despite the precomputed flag), fall back to `AmbientRenderer` without interrupting audio playback. This handles stale precomputed flags.

---

## Audio-Reactive Renderer

Visual language: sharp, energetic, frequency-driven.

FFT data split into three buckets:
- **Bass (20–250 Hz)** → central shape scale / pulse radius. The composition breathes with the kick drum.
- **Mid (250–4000 Hz)** → particle count and velocity. Busier mids = more particles moving faster.
- **Treble (4000–20000 Hz)** → edge brightness / shimmer on geometry. High frequencies add crispness.

Color palette uses the same genre → color family mapping as the ambient renderer (see table below), so the same station always has the same color family regardless of mode. The visual distinction between modes comes from shape and movement language, not color.

GPU-computed via GLSL vertex shaders. Can handle tens of thousands of particles at 60fps, matching Metal visual output.

---

## Ambient Renderer

Visual language: slow, fluid, organic — the opposite of audio-reactive mode.

A Perlin/Simplex noise flow field drives hundreds of slow-moving particles whose trails create aurora/lava-lamp-like effects. The noise field seed drifts slowly over time so the animation never visibly loops.

Station metadata shapes the character:

| Genre | Color palette |
|---|---|
| Jazz | Warm ambers and golds |
| Electronic | Cold blues and purples |
| Classical | Soft greens and creams |
| Rock | Deep reds and oranges |
| Unknown / other | Neutral teal |

Country influences movement speed and particle density as a secondary differentiator — subtle variation so stations feel distinct even in ambient mode.

When a new station loads in ambient mode, the color palette cross-fades to the new station's colors over ~2 seconds while the flow field continues uninterrupted.

---

## Transitions

When any station switch happens:
- Outgoing renderer fades to black over 300ms.
- Incoming renderer fades in over 300ms.
- If the mode changes (audio-reactive ↔ ambient), the visual personality shift happens inside this transition naturally.
- No explicit mode label anywhere on screen. The visual difference is the only signal.

---

## Catalog Precompute Job

A small script walks the radio-browser.info filtered catalog (≥128 kbps, music genre, active) and HEAD-checks each stream URL for CORS headers. Outputs a JSON of station objects with a `corsFriendly: boolean` flag. This becomes the catalog basis for the web app.

The job needs to run periodically (suggested: weekly via GitHub Actions) as station hosting configurations change over time.

---

## Backlog Alternatives

These approaches were considered and rejected for v1 but are preserved if the hybrid approach proves problematic:

**Plan B — Ambient-only**
Drop audio analysis entirely. One ambient visualization for all stations. Full catalog, no CORS dependency, simpler architecture. Loses audio reactivity permanently.

**Plan C — Audio-reactive only**
Only ship CORS-friendly stations. Always audio-synced, no ambient fallback, no mode concept. Clean and consistent but loses ~27% of the catalog.
