# Drifting Web — Design Spec

**Created:** 2026-05-09  
**Status:** In progress  
**iOS counterpart:** `design-spec.md`  
**Port plan:** `plans/2026-05-09-web-port-plan.md`

---

## 1. Why a web version

iOS submission rejected under Guideline 5.2.3. Appeal assessed at ~25–40% probability on first reply — plausible but draining. The web version sidesteps Apple's gatekeeping entirely and ships independently. The iOS app is parked, not abandoned.

All core work carries over: catalog source, direct streaming, recommendation engine, gesture model, catalog filtering. The web version is not a compromise — it is the same product on a different runtime.

---

## 2. What the web version is

Same product principles as iOS:

- **Zero friction to music** — audio starts on first tap; visualization begins immediately
- **The visualization IS the UI** — minimal overlay at edges, unobstructed center
- **Every gesture feels intentional** — swipes are the only navigation
- **The app gets smarter silently** — recommendation engine runs in IndexedDB, never interrupts
- **Local-first** — no account, no login, no backend

---

## 3. Tech stack

| Layer | Choice |
|---|---|
| Framework | React 19 + Vite 6 + TypeScript 5.7 |
| Visualization | Three.js r184 (WebGL) |
| Audio analysis | Web Audio API (AnalyserNode, FFT) |
| Persistence | IndexedDB (recommendation engine, saved stations) |
| Tests | Vitest + Testing Library + jsdom |
| Hosting | TBD — Vercel / Cloudflare Pages / Netlify |
| Domain | TBD |

No backend. All state on-device.

---

## 4. What differs from iOS

| Feature | iOS | Web |
|---|---|---|
| Haptics | Core Haptics, full expressiveness | Not implemented — browser vibration API too limited |
| CarPlay | Supported | Not applicable |
| Lock screen controls | MPNowPlayingInfoCenter (full) | Web Media Session API (partial) — deferred until PWA ships |
| Background audio | Native, always | Requires PWA install on iOS Safari (16.4+); plain Safari pauses on lock |
| App Store | Submitted | Not applicable |
| Visualization | Metal shader (GPU) | WebGL / Three.js GLSL (GPU) — visually equivalent |
| Themes | 5 (Drift, Minimal, Aurora, Neon, Alchemy) | Drift only for now; others deferred |
| Plus / IAP | $4.99 one-time | Web is free; Plus deferred |

---

## 5. Catalog & CORS strategy

**Source:** radio-browser.info public API. Same quality filters as iOS (≥128kbps, music genre, active).

**CORS feasibility test (2026-05-09, 200-station sample):**
- 79% of stations returned CORS headers
- 73% are usable audio streams with CORS
- Geographically broad: DE, FR, NL, US, TR, PL, ES, IT, AT, RU

**Decision: Plan A — full visualization, CORS-filtered catalog**

Catalog entries carry a `corsFriendly` boolean (precomputed at build time). CORS-friendly stations get the audio-reactive WebGL renderer (AnalyserNode FFT); others get the ambient flow-field renderer. No label shown — the visual difference is the signal.

If a CORS-friendly station's analysis returns silence in-browser despite the flag, VisualizationManager falls back to ambient mode after a 2s grace period.

**Catalog build:** `npx tsx scripts/buildCatalog.ts` — outputs `public/catalog.json`. Run periodically to refresh CORS flags as hosting providers change.

**Current state:** `corsFriendly` is forced `false` in `App.tsx` (ambient-only). Re-enable once gesture navigation lands (so users can skip stations that fail silently).

---

## 6. Launch experience

Matches iOS spec §11:

1. Ambient visualization (flow-field shader) starts immediately — no blank frame
2. "drifting" wordmark centered: Cormorant Garamond 300, `clamp(40px, 7vw, 72px)`, `rgba(255,255,255,0.6)`
3. "tap to begin" below: system-ui, 12px, uppercase, `rgba(255,255,255,0.2)`
4. Tap anywhere → "connecting…" replaces subtitle; cursor becomes `wait`
5. Station loads → wordmark fades out over 0.8s (CSS transition); burst animation fires
6. Station info appears bottom-left

**"tap to begin" rationale:** browsers block audio autoplay without a user gesture. The wordmark is the solution, not a workaround — it sets expectation and makes the mandatory gesture deliberate.

---

## 7. Gesture system

Matches iOS spec §5, adapted for touch + mouse on web.

| Gesture | Action | Signal |
|---|---|---|
| Swipe left | Next station | Negative (strength by listen duration) |
| Swipe right | Previous station (history stack) | Mild positive |
| Swipe down | Save / bookmark | Strongest positive |
| Swipe up | Open saved stations shelf | None |
| Tap | Start if idle | None |

- Minimum drag threshold before commit; visual pull indicator; cancel on early release
- History stack: in-memory ordered list of stations in session; not persisted
- "Already saved ♥" toast if station already in favorites (no duplicate added)

**Not implemented yet.** Scheduled for next iteration.

---

## 8. Station info overlay

Always visible while playing. No tap-to-show, no auto-dismiss.

**Bottom-left block:**
- Station name: system-ui, 16px, weight 500, `rgba(255,255,255,0.85)`
- Country: system-ui, 13px, weight 400, `rgba(255,255,255,0.45)` — normalized (see §9)
- Genre: system-ui, 13px, weight 400, `rgba(255,255,255,0.35)` — first tag, title-cased

**Top-right (not yet implemented):** share · settings  
**Bottom-right (not yet implemented):** play/pause

`pointer-events: none` on the info block so gestures pass through to the canvas.

---

## 9. Country name normalization

radio-browser.info uses verbose ISO 3166 title-cased names. Map applied in `App.tsx`:

| Raw | Display |
|---|---|
| The United States Of America | United States |
| The Russian Federation | Russia |
| The United Kingdom Of Great Britain And Northern Ireland | United Kingdom |
| The Netherlands | Netherlands |
| (others as needed) | (short form) |

---

## 10. Visualization system

Matches iOS spec §7, implemented in WebGL (Three.js).

**Renderers:**

| Mode | Trigger | Description |
|---|---|---|
| Ambient | Non-CORS stations or silence fallback | Flow-field FBM shader, genre-seeded color, slow organic motion |
| Audio-reactive | CORS-friendly stations | 8000-particle system driven by AnalyserNode FFT (bass/mid/treble) |

**Visual effects (implemented):**

| Effect | Implementation |
|---|---|
| Burst | Full-screen GLSL shader — smoothstep ring + 8 radial particles, fires on app open and station save |
| Color identity | Genre-seeded color per station, 2s lerp transition between stations |

**Visual effects (not yet implemented):**

| Effect | Notes |
|---|---|
| Ripple rings | Up to 4 simultaneous expanding rings on beat detection (audio-reactive mode) |
| Tap shockwave | Tight ring + echo ring on hold release, intensity scales with duration |
| Tap charge | Contracting ring + center glow during hold |
| Renderer cross-fade | 300ms blend between ambient and audio-reactive — currently instant cut |

**GLSL shader patterns (established — use for all future effects):**

- Center + aspect-correct UV: `vec2 uv = vUv * 2.0 - 1.0; uv.x *= u_aspect;`
- Ring: `smoothstep(0.018, 0.0, abs(dist - radius))` — smooth edge IS the glow
- Ring radius scaled to diagonal: `length(vec2(u_aspect, 1.0)) * 1.1` — fills screen on any viewport
- Fade: `pow(1.0 - phase, 1.5)` — cubic, matches iOS feel
- Blending: `THREE.AdditiveBlending` — brightens underlying visualization
- Phase: when complete, set both `this.phase = -1` AND the uniform to `-1` in the same step — otherwise the shader runs as a full-screen pass every frame

**Performance rules:**

- Pixel ratio capped at `Math.min(window.devicePixelRatio, 1)` — retina would render 4× pixels for no perceptible gain on abstract visuals
- FBM ambient shader: 3 octaves (was 5 — 40% cheaper, visually indistinguishable)
- One `getDelta()` call per frame; track `elapsedMs` manually — `getElapsedTime()` calls `getDelta()` internally as a side effect

---

## 11. Audio

**Player:** `AudioPlayer` wraps `HTMLAudioElement` + `AudioContext` + `AnalyserNode`.

- `play(url, corsFriendly)` — if `corsFriendly`, routes audio through `AudioContext` for FFT analysis; otherwise plays directly without analysis
- `getFrequencyBuckets()` — returns `{ bass, mid, treble }` (0–1 each) for visualization
- `isSilent()` — returns true if all buckets below threshold; used by VisualizationManager silence fallback

**Not yet implemented:**
- Audio fade-in on station start (~1s)
- Audio fade-out on skip (~0.4s)
- Retry on stream failure (up to 2×)
- Load timeout (4s)
- Stall watchdog post-connect
- Auto-resume on network recovery
- Pre-buffering next station while current is playing

---

## 12. Recommendation engine

Matches iOS spec §9. **Not yet implemented.**

Runs entirely in the browser. Persists taste profile and saved stations in IndexedDB.

**Signal weighting:**

| Condition | Signal |
|---|---|
| Skip after <5s | Strong negative |
| Skip 5–30s | Moderate negative |
| Skip >30s | Mild negative |
| Dwell >60s without gesture | Weak positive |
| Swipe right | Mild positive |
| Save | Strongest positive |

**Selection phases:**

1. Cold start (0–15 interactions) — random, genre-diverse, music only
2. Learning (16–75) — weighted random from taste profile
3. Mature (76+) — stronger exploitation (amplified score exponent)

15% exploration (fully random) in all phases. 10% of picks inject a saved station.

---

## 13. Saved stations

Matches iOS spec §5 saved stations behavior. **Not yet implemented.**

- Persisted in IndexedDB
- Save on swipe-down; burst animation triggers (already built)
- "Already saved ♥" toast if duplicate
- Save limit: cap at 25 for now (Plus not planned for web)
- Shelf: up-swipe bottom sheet; tap to re-play; swipe-to-delete
- Each row: station name + country · genre

---

## 14. PWA

Required for background audio on iOS Safari. **Not yet implemented.**

- `manifest.json`: name "Drifting", display standalone, theme color dark, icons
- Service worker: cache shell assets; allow audio continuation in background
- Without PWA install on iOS Safari, audio pauses on screen lock

---

## 15. First-run gesture tutorial

Matches iOS spec §12. Ships alongside gesture implementation. **Not yet implemented.**

- Five hint labels arranged around screen center: ← next · → prev · ↑ favorites · ↓ save · · hold
- Low-opacity white (~0.6), visualization visible underneath
- Shown ~2s after first audio, auto-dismiss on any gesture or after 8s
- Never shown again (persisted in localStorage)

---

## 16. Connectivity handling

**Partially implemented.**

- ✅ Retry up to 3 stations on initial load failure (NotSupportedError, network error)
- ✅ Clean error message on exhausted retries ("Could not connect to a station. Tap to try again.")
- ✅ Navigation failure no longer stops current player (audio keeps playing)
- ❌ 4s load timeout per attempt
- ❌ Stall watchdog post-connect
- ❌ Auto-resume on network recovery

---

## 17. Implementation status summary

| Feature | Status |
|---|---|
| Launch experience | ✅ Done |
| Burst animation | ✅ Done |
| Station info overlay | ✅ Done — favicon tile (genre gradient fallback, fades in on load), now-playing via ICY metadata (CORS stations only, polls every 30s), ♥ badge; name/country/favicon anchored; track title and genre float above/below without shifting |
| Country normalization | ✅ Done |
| Ambient visualization | ✅ Done |
| Audio-reactive visualization | ✅ Built, ⚠️ force-disabled |
| Gesture model | ✅ Done — all four directions + keyboard (←→↑↓ Space) + on-screen chevrons + touch swipe recognition (`useSwipeGesture` hook: 50px H / 80px V thresholds, left-edge dead zone, scroll prevention, pointer capture, flash feedback); hold effects deferred |
| Any-key to start | ✅ Done — any keypress (excluding modifier-only combos) on the launch screen triggers playback |
| Pre-load first station | ✅ Done — on mount, catalog is fetched and first station stream is pre-connected (audio.src set, browser buffers stream); on first tap/keypress, play() reuses the pre-connected player without resetting the buffer |
| Next / prev station | ✅ Done — current audio/UI stays live during transition; "connecting…" shown in center |
| Audio fades | ✅ Done — 1s fade-in on station start; crossfade on navigation (old fades out 400ms while new fades in 1s, triggered only after new stream is ready to eliminate silence gaps) |
| Save + saved shelf | ✅ Done — swipe-down/ArrowDown/↓ saves; swipe-up/ArrowUp/↑ opens shelf; IndexedDB persistence (cap 25); toast feedback; burst on save |
| Play / pause | ✅ Done — Space key; AudioPlayer.pause/resume |
| Recommendation engine | ✅ Done — pure functions, IDB schema (v2), `useRecommendation` hook, and App.tsx wiring all complete; signals recorded on skip/prev/save; dwell timer fires at 60s |
| Info overlay controls (settings) | ✅ Done — share button (copies URL to clipboard), settings gear (opens modal), play/pause toggle; modal has taste profile reset, audio-only toggle (pauses visualization), and clear saved stations |
| Donation jar (Ko-fi link in Settings) | ❌ Not started — add when product is closer to done |
| PWA shell | ❌ Not started |
| First-run tutorial | ✅ Done — `useFirstRunTutorial` hook (2s delay, 8s auto-dismiss, localStorage guard); `GestureTutorial` component; adapts labels for touch (swipe ← next) vs desktop (← prev / → next); hold hint included as placeholder; recommendation tagline "your taste profile builds as you listen"; dismissed by any goNext/goPrevious/save/shelf/playPause/share/audioOnly/settings action |
| Connectivity handling / retry | ✅ Done — full connectivity resilience in `useAudioLifecycle`: 3-attempt retry with 4s load timeout per attempt on initial load and navigation; stall watchdog (5s) auto-advances to next station on stream drop; amber "STREAM DROPPED" toast shows live state ("moving to next station…" while transitioning, "moved to next station" after, tap to dismiss); auto-resume on network recovery (reconnects if stream dropped, resumes if paused); page visibility recovery (resumes paused player on tab focus); all audio/navigation state consolidated in `useAudioLifecycle` |
| Renderer cross-fade (300ms) | ✅ Done — 300ms opacity cross-fade between ambient ↔ audio-reactive renderers; 2s HSL color transition between stations (shortest hue path, smoothstep easing); VisualizationManager owns all color state; station color derived from djb2 hash of station UUID (vivid, deterministic, full wheel coverage — replaces sparse genre map) |
| Genre hue bias on station colors | 🔭 Exploration — shift the hash hue toward a genre-appropriate range (e.g., jazz → warm, electronic → cool) while keeping per-station uniqueness; requires mapping common radio-browser tag variants |
| Tap shockwave / charge effects | ❌ Not started |
| Ripple rings | ❌ Not started |
| Station pre-loading | ✅ Done — preload logic lives inside `useAudioLifecycle`; next station pre-loaded immediately after each station commit; 30s expiry to save bandwidth; re-primes on first pointer activity after expiry; `consume()` in `goNext` skips stream-wait for instant crossfade on forward navigation |
