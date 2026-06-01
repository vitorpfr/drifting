# Spectrale — Web Design Spec

**Created:** 2026-05-09  
**Status:** Live at spectrale.app — pre-release (deep links outstanding)  
**iOS counterpart:** `ios-design-spec.md`  
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
| Hosting | Cloudflare Pages |
| Domain | spectrale.app (live) |

No backend. All state on-device.

---

## 4. What differs from iOS

| Feature | iOS | Web |
|---|---|---|
| Haptics | Core Haptics, full expressiveness | Not implemented — browser vibration API too limited |
| CarPlay | Supported | Not applicable |
| Lock screen controls | MPNowPlayingInfoCenter (full) | ✅ Web Media Session API (PR #31) — correct play/pause state + station-skip on iOS; no scrubber/LIVE label (WebKit limits, see §17) |
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

**Current state:** `corsFriendly` values used as-is from catalog. CORS stations get beat-driven ripples via Web Audio AnalyserNode; non-CORS stations get the ambient 5s pulse timer.

---

## 6. Launch experience

Matches iOS spec §11:

1. Ambient visualization (flow-field shader) starts immediately — no blank frame
2. "spectrale" wordmark centered: Cormorant Garamond 300, `clamp(40px, 7vw, 72px)`, `rgba(255,255,255,0.6)`
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

**Position:** configurable via Settings (corner / center — see §17). Both layouts are visually unchanged from their current implementations; the only change is that the choice is independent of whether the WebGL canvas is on or off.

- **Corner** (default): station info block bottom-left; share · settings top-right; play/pause + volume bottom-right — exactly as today's ambient mode
- **Center**: station name / track title / country · genre stacked at screen center — exactly as today's minimal mode appearance

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
| Ambient (default, shipped) | Always | Flow-field FBM shader, station-seeded color, organic motion; ripple rings are beat-driven on CORS stations (AnalyserNode FFT) or fall back to a 5s autonomous pulse — audio-reactive data enhances the ambient experience but is not required for it |
| Drift (deferred) | Manually enabled via `THEMES_ENABLED` flag | 8000-particle system driven by AnalyserNode FFT (bass/mid/treble); the circle pulses with the beat; deferred to post-launch |

**Ambient renderer parameters (as shipped):**
- Time multiplier: `0.30` (fast, organic)
- Distortion scale: `r * 1.6` (turbulent pattern folding)
- Brightness: `pow(f, 1.2) * 1.55 + 0.12` — floor prevents fully-black areas, reduced peak avoids white bleaching
- Vignette: `smoothstep(0.5, 1.8, dist)` — gentle edge darkening, corners at ~30% brightness
- Station colors: full saturation + brightness (`hsbToRgb(hue, 1.0, 1.0)`)

**Visual effects (implemented):**

| Effect | Implementation |
|---|---|
| Burst | Full-screen GLSL shader — smoothstep ring + 8 radial particles, fires on station change and save |
| Color identity | UUID-seeded color per station (djb2 hash → full HSB wheel), 2s HSL lerp transition |
| Ripple rings | 4-slot expanding rings; CORS stations: beat-driven only (EMA onset detector — bass vs slow baseline α=0.08, threshold 0.04, cooldown 250ms; autonomous timer stopped); non-CORS: 5s autonomous timer; Ambient ring duration: 1.5s (beat-responsive but unhurried) |
| Tap charge | Two-layer center glow (tight core + soft halo) while holding, accumulates over 2s |
| Tap shockwave | Primary ring + echo ring on hold release (~0.7s each); intensity scales with hold duration (max at 2s); 150ms dead zone for taps |
| Volume control | Bottom-right slider (co-located with play/pause); persists across station transitions via `fadeIn(ms, targetVolume)` |

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

- `play(url, corsFriendly)` — if `corsFriendly`, routes audio through `AudioContext` for FFT analysis; otherwise plays directly via `audio.volume` without Web Audio graph
- `getFrequencyBuckets()` — returns `{ bass, mid, treble }` (0–1 each) for visualization
- `isSilent()` — returns true if all buckets below threshold; used by VisualizationManager silence fallback

**Critical invariant — AudioContext is only created for CORS streams.** Creating an AudioContext for non-CORS streams risks starting it in `suspended` state (iOS Safari / Chrome autoplay policy). Once `createMediaElementSource` is called, audio output is re-routed exclusively through the Web Audio graph — a suspended context produces complete silence. Non-CORS streams use `audio.volume` directly and are never connected to a graph.

**Implemented:**
- `fadeIn(durationMs, to = 1)` — 1s fade-in, respects current volume slider setting
- `fadeOut(durationMs)` — 400ms crossfade on skip
- Retry up to 3× with 4s load timeout per attempt
- Stall watchdog (5s) auto-advances to next station
- Auto-resume on network recovery and page visibility restore
- Pre-buffering next station (30s expiry, re-primed on pointer activity)
- `setVolume(v)` — live volume control; `desiredVolumeRef` in `useAudioLifecycle` threads it through all transitions
- Stale buffer reconnect — if the user was paused for >30s, `resume()` clears the src and calls `play()` fresh; on tab return after a long pause, `preconnect()` is called silently in the background so the buffer is ready before the user hits play
- External pause sync — on mobile, any pause that occurs while the app is backgrounded (e.g. lock screen controls) is treated as user-initiated on tab return, preventing unwanted auto-resume

**Mobile preload behavior — `prime()` uses `preconnect()` only on touch devices.** On desktop, the preloaded next-station player calls `play()` silently at volume 0 so the buffer is warm at navigation time. On mobile (`navigator.maxTouchPoints > 0`), only `audio.src` is set (`preconnect()`); `play()` is never called on the preloaded player. This prevents iOS Safari from leaking audio from the preloaded stream — gainNode silencing is unreliable on iOS for MediaElementAudioSourceNode. `play()` is called inside `goNext()` instead, which runs within the swipe gesture context where `AudioContext.resume()` is guaranteed to succeed.

---

## 12. Recommendation engine

Matches iOS spec §9. **Done.**

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

Matches iOS spec §5 saved stations behavior. **Done.**

- Persisted in IndexedDB
- Save on swipe-down; burst animation triggers (already built)
- "Already saved ♥" toast if duplicate
- Save limit: cap at 25 for now (Plus not planned for web)
- Shelf: up-swipe bottom sheet; tap to re-play; swipe-to-delete
- Each row: station name + country · genre

---

## 14. PWA

Required for background audio on iOS Safari. **Not yet implemented.**

- `manifest.json`: name "Spectrale", display standalone, theme color dark, icons
- Service worker: cache shell assets; allow audio continuation in background
- Without PWA install on iOS Safari, audio pauses on screen lock

---

## 15. First-run gesture tutorial

Matches iOS spec §12. **Done.**

- Five hint labels arranged around screen center: ← next · → prev · ↑ favorites · ↓ save · · hold
- Low-opacity white (~0.6), visualization visible underneath
- Shown ~2s after first audio, auto-dismiss on any gesture or after 8s
- Never shown again (persisted in localStorage)

---

## 16. Connectivity handling

**Done.** See §17 status entry for full detail.

---

## 17. Implementation status summary

| Feature | Status |
|---|---|
| Launch experience | ✅ Done (PR #21) — full-screen idle phase eliminated; catalog loads on mount and first station preconnects before any tap; wide frosted-glass `LandingCard` banner (bottom of screen, 80px above edge) shows logo + tagline + "tap to play / N+ radio stations worldwide"; shimmer skeleton visible behind card (animated, no station name — avoids promising a specific station); banner z-index split: background handler at z8 (below UI controls) + card at z50 so all buttons remain clickable during landing; gesture gate (`gestureReceivedRef` + `gestureWakeRef`) blocks `play()` until `releaseGesture()` fires synchronously from tap handler; after tap card fades (600ms), skeleton animates, "connecting…" appears at same position until station commits; `StationInfo` renamed `CornerStationInfo` with `isConnecting`/`retryMessage` props matching `CenteredStationInfo`; `hasStarted` boolean replaces `'idle'` phase throughout — `Phase` type is now `loading \| exiting \| playing`; deep link: preconnects the linked station (not a random one), retries only that station on failure; favicon skeleton uses same shimmer as text skeletons; quality filters applied to preconnect pool to match `loadInitial` pool; landing card no longer shows random station name — `isDeepLink` prop gates station name display (shown only for deep link sessions) |
| Startup station mode | ✅ Done — "First station: Auto / From saved" row in Settings with description "Which station plays first when you open the app."; `startupStationMode: 'auto' \| 'saved'` in `UIPreferencesContext` (localStorage `startup_station_mode`, default `'auto'`); `getStartupStation` callback option in `useAudioLifecycle` called once in `loadInitial` before the preloaded station is used; falls back to `auto` silently when saved list is empty; deep link always overrides; retry after failed startup station falls back to `pickNext`; Settings modal panel gained `maxHeight: 85vh` + `overflowY: auto` to prevent overflow on short viewports |
| Burst animation | ✅ Done |
| Station info overlay | ✅ Done — favicon tile (genre gradient fallback, fades in on load), now-playing via ICY metadata (CORS stations only, polls every 10s; null results keep previous title; two-attempt fetch: first with `Icy-MetaData: 1` for positioned mode, fallback without header to avoid CORS preflight rejection — scans body bytes for `StreamTitle=` pattern; stops polling after 3 consecutive failures to avoid console spam for HTTP-only stations blocked by browser HTTPS policies), ♥ badge; name/country/favicon anchored; track title and genre float above/below without shifting |
| Country normalization | ✅ Done (PR #18) — shared `src/countryNames.ts` with 29-entry map; applied at build time in `buildCatalog.ts`; fixes `'russian'`→`'russia'` and `'turkey'`→`'türkiye'` locale bias bugs; `RESTRICTED_COUNTRIES` updated to normalized names |
| Ambient visualization | ✅ Done |
| Audio-reactive visualization (ambient) | ✅ Active — ripple rings in Ambient theme are beat-driven via AnalyserNode FFT on CORS stations; falls back to a 2s autonomous pulse when CORS is unavailable or AnalyserNode returns silence; on iOS Safari the AnalyserNode returns all zeros for cross-origin streams (WebKit CORS taint) — detected after 3s of zero data and switched to the autonomous pulse; non-CORS stations always use the autonomous pulse at 2s |
| Gesture model | ✅ Done (PR #18 update) — all four directions + keyboard (←→↑↓ Space S F B V , . ? [ ]) + on-screen chevrons + touch swipe recognition (`useSwipeGesture` hook: 50px H / 80px V thresholds, left-edge dead zone, scroll prevention, pointer capture, flash feedback); `[` / `]` adjusts volume ±10% (clamped, desktop only); on mobile the bottom button cluster (Search/Filter/Block/Play) sits at bottom edge, down arrow floats above it at 72px; button gap unified to 20px matching top-right cluster; hold effects deferred |
| Any-key to start | ✅ Done — any keypress (excluding modifier-only combos) on the launch screen triggers playback |
| Pre-load first station | ✅ Done — on mount, catalog is fetched and first station stream is pre-connected (audio.src set, browser buffers stream); on first tap/keypress, play() reuses the pre-connected player without resetting the buffer |
| Next / prev station | ✅ Done — current audio/UI stays live during transition; "connecting…" shown in center |
| Audio fades | ✅ Done — 1s fade-in on station start; crossfade on navigation (old fades out 400ms while new fades in 1s, triggered only after new stream is ready to eliminate silence gaps) |
| Save + saved shelf | ✅ Done — swipe-down/ArrowDown/↓ saves; swipe-up/ArrowUp/↑ opens shelf; IndexedDB persistence (cap 25); toast feedback; burst on save; toast is dismissed when shelf opens (prevents z-index overlap) |
| Play / pause | ✅ Done — Space key; AudioPlayer.pause/resume |
| Recommendation engine | ✅ Done — pure functions, IDB schema (v2), `useRecommendation` hook, and App.tsx wiring all complete; signals recorded on skip/prev/save; dwell timer fires at 60s |
| Info overlay controls (settings) | ✅ Done — share button (copies URL to clipboard), settings gear (opens modal), play/pause toggle; modal has taste profile reset, audio-only toggle (pauses visualization), and clear saved stations |
| Minimal mode | ✅ Done (PR #17) — controls background only: eye icon (top-right, between ? and Settings) / `V` key toggles WebGL canvas on/off; canvas off = dark matte screen (`#08080d`) with ~12% station color radial wash; canvas fades out (600ms manual toggle, 3.5s from landing); visualization always running in background when canvas is off; audio activity badge (animated bars / muted / static dot) shown in both modes; favicon shown in connecting state; top-right controls hidden during landing/idle phase. Minimal mode is background-only — it does not affect station info layout position (that is a separate, independent setting). **Tab bar (PR #33):** in minimal mode the Safari `theme-color`/body background is tinted to the matte `#08080d` (shared `MINIMAL_BACKGROUND`) instead of the station color, matching the screen; transitions smoothly back to the station color on exit via the body's 2s `background-color` ease (macOS reads the CSS bg; iOS reads the meta tag and jumps, consistent with existing station-change behaviour) |
| Donation jar (Ko-fi link in Settings) | ✅ Done — "Support Spectrale / Buy a coffee ☕" row at bottom of settings modal; links to ko-fi.com/spectrale |
| First-run tutorial | ✅ Done — `useFirstRunTutorial` hook (3s delay, 5s auto-dismiss, localStorage guard); `GestureTutorial` component; adapts labels for touch (swipe ← next) vs desktop (← prev / → next); hold hint included as placeholder; recommendation tagline "your taste profile builds as you listen"; desktop-only "? for all shortcuts" hint at bottom; "↓ save" label positioned lower on desktop (`bottom: 16%`) vs mobile (`bottom: 28%`) to avoid overlap with centre content; no longer gated by genre seed completion — shows independently at 3s; dismissed by any goNext/goPrevious/save/shelf/playPause/share/audioOnly/settings action |
| Connectivity handling / retry | ✅ Done — orchestrator pattern in `useAudioLifecycle`: both `loadInitial` and `goNext` retry indefinitely (while-true loop, 5s per attempt) until a station connects; each failure passes the failed station as `lastAttempted` to `pickNext` so every retry picks a different station; second tap while connecting is a no-op (orchestrator guarantees a station will land); brief "Connection failed, trying next station…" flash shown during `goNext` retries (silent on initial load); stall watchdog (5s) auto-advances to next station on stream drop; amber stall toast ("stream dropped, moving…" / "moved") shown only for watchdog-triggered nav, not user-initiated; session-level `AudioContext` singleton (`sessionAudioContext.ts`) shared across all `AudioPlayer` instances so iOS Safari never needs to resume it again after the first gesture; auto-resume on network recovery; page visibility recovery |
| Renderer cross-fade (300ms) | ✅ Done — 300ms opacity cross-fade between ambient ↔ audio-reactive renderers; 2s HSL color transition between stations (shortest hue path, smoothstep easing); VisualizationManager owns all color state; station color derived from djb2 hash of station UUID — three tiers: ~75% colorful (full saturation/brightness), ~12.5% grey/silver (low saturation, full brightness), ~12.5% dark/matte (low brightness 0.30–0.45, near-zero saturation 0.08); tier and hue are deterministic per station UUID |
| Genre hue bias on station colors | 🔭 Exploration — shift the hash hue toward a genre-appropriate range (e.g., jazz → warm, electronic → cool) while keeping per-station uniqueness; requires mapping common radio-browser tag variants |
| Ko-fi nudge | ✅ Done — fires after "Saved ♥" toast dismisses once 10 cumulative saves threshold is reached (tracked in `saves_count` localStorage); once per session + 7-day cooldown; auto-dismisses after 5s; copy: "Enjoying Spectrale? Support development ☕"; positioned above station info on mobile |
| Auth-required stream blocking | ✅ Done — fetch() HEAD probe before audio element load; 401 → treated as failed stream, retry logic skips to next station; probe skipped on mobile (touch devices) to preserve iOS Safari autoplay gesture context and avoid consuming the 4s load timeout budget |
| HQ quality badge | ✅ Done — "HQ" shown next to station name for ≥192kbps streams (~17% of catalog); replaces debug BEAT/PULSE badge |
| Hosting + domain | ✅ Done — Cloudflare Pages; spectrale.app live (2026-05-17); Ko-fi donation link operational (Stripe integrated) |
| Per-station deep link + share button | ✅ Done — share button copies `?station=<uuid>` when playing; deep link auto-plays that station on load; URL cleaned up after first station loads; falls back to random pick if UUID not in catalog; launch screen shows "tap to play [Station Name]" when opened via deep link; idle subtitle updated to "tap to discover radio in color". **Unloadable deep-link station (PR #33):** retried up to 4× with a 700ms backoff (an explicit request gets more chances than a plain "play next"), then falls back to `pickNext` — previously a failing linked station looped forever on CONNECTING because `linkedStation` was reused on every retry iteration |
| PWA shell | ✅ Done — `manifest.json` (standalone display, dark theme, icons from existing 1024×1024 favicon); minimal service worker (network-first navigation, offline fallback to cached index); iOS meta tags (`apple-mobile-web-app-capable`, touch icon, status bar); registered in `main.tsx` on load |
| Tap shockwave / charge effects | ✅ Done — `ShockwaveEffect`: center glow charge, primary + echo ring on release; hold >150ms triggers; intensity 0→1 over 2s hold; `App.tsx` pointer handlers with 15px movement cancel |
| Ripple rings | ✅ Done — `RippleEffect`: 4-slot rings; CORS stations beat-driven (EMA onset detector, threshold 0.04, cooldown 250ms); non-CORS 2s autonomous pulse; CORS stations use beat detection on desktop and fall back to 2s autonomous pulse on iOS (AnalyserNode returns zeros); Ambient ring duration 1.5s (`RING_SPEED_AMBIENT = 1/1.5`), Drift 3s; ring edges softened (smoothstep 0.030 vs original 0.012) and deltaMs capped at 50ms for smoother appearance at 60fps; ripples gated on playing state via `VisualizationManager.setPlaying()` — no rings on landing screen, during loading, or while paused; single ring fires on resume from pause |
| Volume slider | ✅ Done (PR #18 update) — bottom-right, co-located with play/pause; 44px touch target; persists across station transitions and sessions (localStorage); saved volume applied from first station load on session start (not just after first slider interaction); hidden on touch/mobile (device volume controls suffice) |
| SEO foundations | ✅ Done — keyword-rich meta description and OG/Twitter tags; canonical link; JSON-LD WebApplication schema; robots.txt; sitemap.xml; noscript static content for crawlers |
| og:image (social preview card) | ✅ Done — 2400×1260 @2x branded card (wordmark, tagline, ambient glow blobs, ripple rings); wired in og:image + twitter:image meta tags |
| Privacy policy | ✅ Done — `spectrale.app/privacy.html`; covers IndexedDB/localStorage, Radio Browser catalog requests, Umami analytics; linked from Settings modal footer |
| Send feedback link | ✅ Done — `hello@spectrale.app` mailto link in Settings modal |
| Accessibility / screen reader support | 🟡 Mostly done (PR #15) — `aria-live` announcer for station changes; `useFocusTrap` in all 5 modals (Settings, Filter, GenreSeed, SavedShelf, Search); `aria-labelledby` on all dialogs; `aria-pressed` on toggle chips; favicon alt text; `:focus-visible` global style; roving tabindex in FilterModal genre chips (arrow keys); keyboard shortcuts modal (`?` key + `?` button); shortcuts for all major actions (← → ↑ ↓ Space S F , . ?); descriptive `aria-label` on Settings Reset and Clear buttons. Known gaps: VoiceOver navigation inside ShortcutsModal rows is unreliable (table/kbd structure added but not fully verified); GenreSeed uses plain tab order (arrow keys conflicted with App global handler during idle phase) |
| Analytics instrumentation | ✅ Done (PR #18 update) — `analytics.ts` wrapper (`trackEvent`); Umami Cloud (EU region, cookie-free, no consent popup required); script loaded via `index.html`; 6 events: `playback_started`, `playback_duration`, `station_skipped`, `station_saved`, `station_blocked`, `genre_seed_completed`; console.log in dev, `window.umami.track()` in prod |
| Ambient renderer improvements | ✅ Done — speed 0.30, turbulence r*1.6, brightness floor +0.12, vignette softened, station colors full saturation |
| Station pre-loading | ✅ Done — preload logic lives inside `useAudioLifecycle`; next station pre-loaded immediately after each station commit; 30s expiry to save bandwidth; re-primes on first pointer activity after expiry; `consume()` in `goNext` skips stream-wait for instant crossfade on forward navigation; desktop: pre-plays at volume 0 (buffer + AudioContext warm); mobile: preconnect() only — play() is deferred to goNext() inside the swipe gesture to prevent iOS audio leakage |
| Build SHA fingerprint | ✅ Done — `CF_PAGES_COMMIT_SHA` injected at build time via Vite `define`; shown in Settings modal footer (faint, selectable) and logged to browser console on startup; `'local'` fallback in dev |
| Search stations | ✅ Done — search icon in bottom-right controls; `S` key shortcut (desktop); multi-word search (splits on whitespace, all words must match); arrow keys navigate results, Enter plays; centered panel on desktop (480px), fullscreen on mobile |
| Filter stations | ✅ Done — filter icon in bottom-right controls; `F` key shortcut (desktop); genre chip picker + country dropdown; arrow keys navigate genre chips (Left/Right), Enter/Space toggles; centered panel on desktop (400px), fullscreen on mobile; active filter persists until cleared |
| MediaSession API | ✅ Done (PR #31 `fix/ios-now-playing`) — Control Center / lock-screen Now Playing works on iOS: correct **Pause** state for every station, and station-skip controls. **Root cause of the original bug** (station 1 showed Pause, stations 2+ showed Play despite playing): iOS Safari ties the system media session to the single `HTMLAudioElement` it auto-detects when playback first starts. The crossfade architecture created a fresh element per station, so iOS only ever tracked station 1's element; stations 2+ played on new elements iOS never adopted, leaving Control Center reporting against the dead/paused first element. Confirmed on macOS Safari too — not iOS-specific buffering/volume. **Fix**: on the `skipWebAudio` path (iOS/iPadOS, touch Macs) all `AudioPlayer`s share ONE `HTMLAudioElement` (`sharedMediaElement`) and just swap `src` on navigation; an ownership guard (`sharedOwner` / `ownsElement()`) makes the outgoing player's deferred `fadeOut`/`stop` no-ops so they can't clobber the element the incoming player just claimed. Desktop keeps its per-element two-element crossfade. **WebKit constraints found (documented in code, on-device verified)**: (1) the Apple-Music "LIVE" label is unreachable — `setPositionState({duration: Infinity})` throws `non-finite`; (2) the seek-vs-track button choice (±10s skip vs ⏮/⏭) is a non-deterministic race latched at session establishment and immutable from JS afterward, so we map `seekforward`/`seekbackward` to next/previous-station so whichever buttons iOS shows can skip stations; (3) a smooth audio crossfade is impossible on iOS — `audio.volume` is ignored from JS and the only alternative (Web Audio `GainNode`) requires `createMediaElementSource`, which detaches the media session and reintroduces the bug — so iOS transitions are a hard cut. |
| Genre seed on first launch | ✅ Done (redesigned) — appears at 15s after playing starts (after tutorial completes at 8s, giving 7s of uninterrupted listening); single-screen modal with intro tagline "Spectrale discovers radio for you. The more you skip and save, the smarter it gets."; mode selector: `[Personalize]` (tailored picks, **default**) vs `[Random]` (stations picked at random from anywhere in the world); reassurance text "You can change this anytime in Settings"; genre chips shown below, dimmed/non-interactive in Random, active in Personalize (max 3, optional); CTA is "Start listening →" for both modes; mode saved to `localStorage('discovery_mode')`, default `'personalize'`; signals recorded in both modes; resetting taste profile also clears `genre_seed_shown` so modal re-appears; Settings gets "Station selection" row (`[Personalized]` / `[Random]`) with description "How next stations are picked as you listen."; "Clear saved stations" moved out of Settings into the Saved shelf (see Saved shelf row) |
| Saved shelf improvements | ✅ Done — animated equalizer on playing station row; sort chips (Saved · A–Z · Country · Genre, persisted to localStorage); drag-to-reorder in Saved mode (HTML5 DnD desktop + touch); ArrowUp/Down opens/closes shelf; 1–0 keys play first 10 stations with visible number badges (desktop only); "Clear all" button in shelf header (visible in Saved view when stations exist) with inline confirmation panel ("Remove all N saved stations? / Cancel / Clear all") before executing; removed from Settings |
| Listening stats | ✅ Done — `totalListenSeconds` added to TasteProfile (zero-safe for existing profiles), accumulated in `onNavigatingAway`; `getStats()` in `useRecommendation` derives top genres + top countries from existing positive scores + total time + interaction count; "Your listening" section in Settings modal shows time / stations / top genres / top countries when `interactionCount > 0` |
| Sleep timer | ✅ Done — 30m / 60m / 2h picker in Settings (pill buttons, tapping active duration cancels); `useSleepTimer` hook owns countdown and expiry; 400ms fade then pause on expiry; countdown pill indicator (`⏾ Xm`) in top-right controls cluster, tap to cancel; cancelled on manual pause; timer survives station navigation; does not persist across page reloads |
| HQ-only filter toggle | ✅ Done — Settings toggle (On/Off buttons, same style as station info position) restricts catalog to ≥192kbps streams (~17% of catalog, matching HQ badge threshold); persisted to localStorage (`hq_only`); toast shown on toggle while playing ("HQ only — applies from next station →" / "HQ filter off"); wired into `filterCatalog` via `UIPreferencesContext` |
| No-connection overlay | ✅ Done — `isOffline` state in `useAudioLifecycle` (init from `navigator.onLine`, updated by `online`/`offline` events); overlay shows "No connection / Will resume automatically" centered on screen; clears automatically on `online` event which also resumes playback |
| Persistent listening history | ✅ Done — IDB v3 `listenHistory` store (keyPath: `stationuuid`, upsert on commit); `useListenHistory` hook loads on mount and exposes `addToHistory`/`clearHistory`; max 30 entries, most-recent-first; saved shelf gains a "History" chip that switches to the history list (read-only, tap to play); wired in App.tsx via `onStationChanged` |
| Failing saved station indicator | 🔭 Deferred — ⚠ badge on saved station rows that failed to connect; dropped from initial scope because the existing retry UX already handles failure gracefully; background probing would burn bandwidth for marginal gain |
| First-station quality boost | ✅ Done (PR #18 update) — window extended 3→15 picks; filter now prefers `streamLive: true` stations (verified alive at build time) if ≥10 available; falls back to favicon+bitrate quality pool; stream verification runs as second pass in `buildCatalog.ts` (3s timeout, audio content-type check, concurrency 20); catalog build switched to `all.api.radio-browser.info` (load-balanced) and 64kbps floor (was 128kbps); current catalog: 14,119 stations across 176 countries, 8,370 stream-verified |
| Locale-aware cold start | ✅ Done — `detectLocaleCountry()` in `utils.ts` maps device timezone (via `Intl.DateTimeFormat`) to a catalog country fragment (~30 IANA zones, prefix fallback for `America/` and `Australia/`); `localeBiasFilter()` narrows the pool to locale-matching stations for the first 3 picks, falls back to full pool if fewer than 10 local stations match; wired alongside `qualityBoostFilter` in `prime` and `loadInitial` |
| Drift theme | 🔭 Deferred — infrastructure complete (AudioReactiveRenderer GLSL shader, beat detection, crossfade, `THEMES_ENABLED` flag in `src/config/features.ts`); works on desktop (confirmed visually); on iOS Safari the AnalyserNode returns all zeros so the beat-driven circle pulsation is absent there; suspected cause: WebKit CORS taint or AnalyserNode quirk; next step when revisiting: add a debug readout of `getByteFrequencyData` values on iOS to confirm whether any non-zero data ever arrives; intentionally deferred to post-launch — not a blocker for the Ambient theme |
| Station blocking | ✅ Done (PR #18) — B key + ban icon button in OverlayControls bottom cluster; IDB v4 `blockedStations` store; `useBlockedStations` hook; excluded from `filterCatalog` + `getSavedStations`; 5s undo toast with Undo button (calls `goPrevious()`); Blocked tab in Saved Shelf (hidden until non-empty, playable rows + Unblock button); saved/blocked mutually exclusive |
| Visualization themes | ✅ Done — PR #12 merged; `Theme` type ('ambient' \| 'drift'); `THEMES_ENABLED` flag in `src/config/features.ts` (currently `false`); `VisualizationManager.setTheme()` switches renderer via existing 300ms crossfade; Ambient = FBM flow-field shader (default); Drift = audio-reactive particle/ring shader; Drift + no CORS falls back to breathing ring naturally (zero audio input → sin-based oscillation only); remaining themes (Minimal, Aurora, Neon, Alchemy) not yet started |
| Station info position | ✅ Done (PR #19) — Corner/Center pill toggle in Settings (Center first, Center is default); persisted to `localStorage`; fully independent of minimal mode; `CenteredStationInfo` component renders station name, country·genre, now-playing, favicon, and shimmer skeletons when loading; favicon hidden (opacity 0, not removed) during initial load to avoid conflicting with the energy ball while preserving layout so text positions never shift; connecting text repositioned below center content (above nav arrow) in center mode, stays at screen center in corner mode; connecting text readability matched to landing CTA (fontSize 14, fontWeight 500, rgba(255,255,255,0.65)); `StationInfoSkeleton` unified with `centered` prop — one component, two layouts; `UIPreferencesContext` centralizes `isMinimalMode`, `isStationInfoCentered`, `currentTheme` with localStorage persistence, eliminating prop-drilling from App.tsx through OverlayControls/SettingsModal/Toast/KofiNudge; `isTouch` consolidated from 8 duplicate declarations into `utils.ts` |
| Performance / adaptive quality | ✅ Done (PR #20) — three-layer approach: Layer 0 (synchronous, pre-render): `UIPreferencesContext` `useState` initializer checks `prefers-reduced-motion` and `hardwareConcurrency ≤ 2 + deviceMemory ≤ 1`; if triggered, `isMinimalMode` starts `true` before first frame, viz shader suppressed at mount via `initialMinimalModeRef`; Layer 1 (FPS sampling): `useAdaptiveQuality` hook samples 45 rAF frames during idle, computes median inter-frame delta (robust to GC spikes), triggers silent minimal mode + 600ms canvas fade if median FPS < 28 — even during idle phase, bypassing the normal `phase !== 'idle'` guard; both layers write `minimal_mode = true` to `localStorage` so return visits skip re-check; full top-right control row (Share · ? · Eye · Settings) always visible during landing (not just during playback) so any user can manually toggle before playing; 3-ring CSS ripple burst on first station load in minimal mode (respects `prefers-reduced-motion`); WebGL mode fires matching 3 staggered ripple rings on every station change; smooth 400ms crossfade in both directions when toggling ambient↔minimal while playing (always-rendered `MinimalScreen` enables fade-in); HQ badge consistent in both corner and centered station info; filter change discards preloaded station and shows toast "Filter applies from next station →"; GenreSeed popup delay increased to 20s |
| Snapshot share card | ✅ Done — `VisualizationManager.captureFrame()` (synchronous `tick()` + `toDataURL` in same tick, no `preserveDrawingBuffer`); `src/shareCard.ts` with `computeCardLayout`, `drawShareCard` (1080×1080 canvas: live frame background, bottom scrim, station color dot, title/subtitle, "spectrale" wordmark), `renderShareCardBlob`, and `shareStation` orchestrator with injected deps; Share button: on mobile only attempts image share when `navigator.canShare({files})` returns true (HTTPS required) — native share sheet with image; all other cases (desktop, HTTP, canShare false) fall back to instant clipboard-copy; `station_shared` analytics event with `method: 'image' \| 'link'`; `scripts/findDemoStations.ts` outputs deep links to best CORS + high-bitrate + colorful-tier stations for demo video recording |
| PWA Add to Home Screen prompt | ✅ Removed — prompt was in Settings (iOS Safari only, not already installed); removed in PR #29 because the dynamic theme-color tab bar makes Safari's browser chrome feel integrated with the visualization, making PWA install no longer a meaningful upgrade; PWA manifest/service worker still ship for completeness but are not advertised |
| Safari/desktop visualization polish | ✅ Done (PR #29) — dynamic `theme-color` meta tag + `document.body.backgroundColor` updated on each station change so Safari 15+ colors the tab bar to match the visualization (darkened to L=0.25, 2s ease-in-out CSS transition in sync with the viz color transition, instant on landing); `will-change: transform` on WebGL canvas promotes it to its own compositor layer; `powerPreference: 'high-performance'` on WebGLRenderer hints Safari to use discrete GPU; FBM ambient shader hash replaced with sin-free multiplicative hash (60× fewer transcendental calls per fragment); ambient brightness reduced (playing max 1.50× vs original 2.30×); favicon fallback state keyed to station UUID (prevents race where loaded state from previous station persisted to next); false stream drop fix (stall watchdog now also cancels on `timeupdate` — iOS sometimes doesn't re-emit `playing` after brief buffering); desktop share button always copies link (was incorrectly triggering image share sheet on macOS Safari via `navigator.canShare`) |
| Per-station OG images | ✅ Done — `scripts/buildOgIndex.ts` generates `functions/og-index.json` (slim `uuid → {n,c,g}` map, 14,119 stations); `functions/og.ts` renders a 1200×630 PNG with `workers-og` (Satori/resvg), accent color from `getStationColor` so it matches the live app, station name/genre from the index, color-only fallback for unknown UUIDs, `Cache-Control: immutable`; `functions/_middleware.ts` uses HTMLRewriter to rewrite `og:title`, `twitter:title`, `og:image`, `twitter:image`, `og:url` for `?station=<uuid>` requests — zero overhead when no station param, passes through non-HTML responses untouched; `src/ogMeta.ts` pure tag-builder unit-tested independently; documented as the single justified exception to the no-backend principle (functions touch no user state); run `npm run build:og-index` and commit when catalog is rebuilt |
| Station info text legibility | ✅ Done (PR #32) — country name: opacity 0.45→0.65, fontWeight 400→500 (corner); genre: 0.35→0.55, 400→500 (corner); country·genre combined line: 0.35→0.55, 12px→13px, added fontWeight 500 (centered); now-playing track title: 0.55→0.65 (corner), 0.45→0.60 (centered) — all were too dim to read against the visualization on mobile |
| Search → play NotSupportedError on iOS (Oroko) | 🟡 Investigated & mitigated, not deterministically fixable (PR #33) — the earlier `\|\| readyState === 0` src-reset "fix" was **reverted**: on-device evidence (flag-gated debug overlay capturing `audio.error.code`/`networkState`) showed it *regressed* the deep-link path by discarding the warm preconnect buffer (`audio.pause()` drops `readyState` to 0 → clause reset `src` → cold reload → fail). **Findings**: the bug is **station-specific and non-deterministic** — a plain MP3 station plays fine on the same cold `playStation` path; Oroko fails with `code=4 / net=3` (`MEDIA_ERR_SRC_NOT_SUPPORTED`), yet `curl` shows the stream healthy (206, valid 320kbps MP3, `ACAO:*`). Deep-link (warm preconnect) plays it on `main` but the same code/stream sometimes fails minutes later — i.e. an **iOS-Safari-media-session × Icecast-stream interaction we don't control**, not a fixable line of app code. **Shipped instead** (graceful degradation): deep-link bounded-retry-then-fallback (no more infinite CONNECTING); "Couldn't connect to {name}" feedback toast for failed user-initiated picks; retry-message parity in centered layout. The `?debug=1` overlay (prod-disabled) remains in-tree for future on-device validation. |
| Visibility-aware transient UI | ✅ Done (PR #33) — shared `useVisibilityAwareTimeout` hook: auto-dismiss countdown only elapses while the page is visible (pauses when the tab is backgrounded / phone locked, resumes the remainder on return; re-arms per `restartKey`, always calls the latest callback). Applied to `Toast` (now visibility-aware by default — the per-toast opt-in was removed), `KofiNudge` (5s), and the stall notice ("stream dropped, moving…/moved", switched from click-dismiss to a 4s timer). Keeps transient messages from being spent before the user sees them |
| Failed-pick feedback toast | ✅ Done (PR #33) — `onPlayFailed` callback from `useAudioLifecycle`: when a user-initiated pick (search / saved shelf / number keys) fails every attempt, show "Couldn't connect to {name}" (4s, visibility-aware) while the current station keeps playing — no random fallback for an explicit request |
| Debug overlay (on-device) | ✅ Done (PR #33) — `?debug=1` enables an in-screen log overlay (with copy button) on preview deploys + localhost (incl. iOS, no console needed; persisted to localStorage, `?debug=0` to disable); **hard-disabled on the production host** (`spectrale.app`) regardless of URL. `debugLog()` and the overlay no-op when off; near-zero cost in production. Kept in-tree for quick future on-device investigations |

---

## 18. Pre-launch roadmap

Features to complete before posting to Reddit / driving external traffic. Ordered by priority.

### Must-have before posting

| Feature | Why |
|---|---|
| og:image | ✅ Done — Reddit and social link previews are blank without it — single biggest conversion factor for a click |

### High value, ship before or shortly after posting

| Feature | Why |
|---|---|
| Drift theme | Beat-driven circle pulsation is the visual centrepiece; infrastructure complete, iOS Safari AnalyserNode issue to investigate before enabling (see status row above) |

### Post-launch, based on feedback

| Feature | Notes |
|---|---|
| Performance / adaptive quality | Protects first impression on low-end devices; `prefers-reduced-motion` support is also an accessibility gap |
| Sleep timer | Common radio app request; straightforward to build |
| HQ-only filter toggle | ✅ Done (PR #24) |
| Failing saved station indicator | 🔭 Deferred — existing retry UX handles failure gracefully; background probing has poor cost/benefit |
| Visualization themes | Infrastructure merged (PR #12); `THEMES_ENABLED` flag off; remaining themes (Minimal, Aurora, Neon, Alchemy) not yet started |

---

## 19. Growth / distribution initiative (2026-05-30)

The app is feature-complete but distribution-starved — one user, no advertising. The bottleneck is not more features; it is (a) the product not yet spreading itself and (b) no launch channels exercised. This initiative addresses both. **Planned, not yet implemented — next session implements.**

> **▶ Immediate next step (next session): capture the demo video + screenshots.** With iOS Now Playing (PR #31) shipped, the next concrete task is recording the 12–18s silent looping demo video and the screenshots — the master asset that gates Uneed/Product Hunt and lifts every other channel. Everything downstream waits on these assets.

**Product side (code — see §17 "Snapshot share card" and "Per-station OG images"):** Spectrale's value is visual and per-station-unique, yet the only share mechanism is a text URL. The two planned features close that gap — a shareable image of the live visualization, and station-specific link-unfurl previews. Spec: `docs/superpowers/specs/2026-05-30-shareable-cards-design.md`. Plan: `docs/superpowers/plans/2026-05-30-shareable-cards.md` (two independently-shippable parts; ship Part A / snapshot first to validate before edge functions).

**Distribution side (assets + copy, no code):** Draft plan at `docs/superpowers/plans/2026-05-30-distribution-assets.md`. Master asset is a 12–18s silent looping demo video (gates Uneed, Product Hunt, and lifts every other channel). Priority channels: r/InternetIsBeautiful (best single fit, one-shot), Show HN (technical angle: client-side reco engine, no backend, WebGL FFT), Uneed (draft already waiting on the video), directories (AlternativeTo et al.), Product Hunt (later). Sequence the high-value posts *after* the share features ship so inbound traffic spreads itself.
